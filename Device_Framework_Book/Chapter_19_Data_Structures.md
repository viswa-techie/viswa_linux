# Chapter 19: Framework Data Structures Deep Dive

## Learning Goals
- Understand the key data structures that power each framework
- Know how structures are linked: embedding, container_of, lists
- Trace structure relationships from device model to framework-specific types
- Analyze data structure layout for debugging and performance

---

## 19.1 Core Device Model Structures

```
Device Model — Structure Hierarchy:

struct bus_type          struct device_driver         struct device
┌─────────────────┐    ┌────────────────────┐    ┌─────────────────────┐
│ name = "i2c"    │    │ name = "tmp102"    │    │ bus = &i2c_bus_type │
│ match()         │◄───│ bus = &i2c_bus_type│    │ driver = &tmp102_drv│
│ probe()         │    │ probe()            │    │ of_node             │
│ uevent()        │    │ of_match_table     │    │ parent              │
│ drivers (list)  │    │ pm = &pm_ops       │    │ devres_head (list)  │
│ devices (list)  │    └────────────────────┘    │ power (dev_pm_info) │
└─────────────────┘                              │ kobj (kobject/sysfs)│
                                                 │ platform_data      │
                                                 │ driver_data (priv) │
                                                 └─────────────────────┘

Embedding pattern — framework structures contain struct device:

struct platform_device {
    const char *name;
    struct resource *resource;  ← MMIO, IRQ, DMA
    struct device dev;          ← embedded (not pointer)
};

struct i2c_client {
    unsigned short addr;        ← 7-bit slave address
    struct i2c_adapter *adapter;← which I2C bus
    struct device dev;          ← embedded
};

struct spi_device {
    u32 max_speed_hz;
    u8 mode;                   ← CPOL/CPHA
    u8 bits_per_word;
    struct device dev;         ← embedded
};

/* container_of recovers the parent from embedded struct */
static int my_probe(struct device *dev)
{
    struct platform_device *pdev = to_platform_device(dev);
    /* Expands to: container_of(dev, struct platform_device, dev) */
}
```

---

## 19.2 V4L2 Data Structures

```
V4L2 Structure Map:

struct v4l2_device (one per hardware instance)
├── dev = &pdev->dev
├── name = "my-camera"
├── subdevs (list of v4l2_subdev)
└── mdev (media_device pointer)
     │
     ├── struct media_device
     │   ├── model = "My Camera"
     │   ├── entities (list of media_entity)
     │   └── /dev/media0
     │
     ├── struct video_device
     │   ├── fops (v4l2_file_operations)
     │   ├── ioctl_ops (v4l2_ioctl_ops)
     │   ├── queue (vb2_queue pointer)
     │   ├── v4l2_dev pointer
     │   ├── entity (media_entity)
     │   └── /dev/video0
     │
     └── struct v4l2_subdev
         ├── ops (v4l2_subdev_ops)
         │   ├── video_ops (s_stream)
         │   └── pad_ops (set_fmt, get_fmt)
         ├── entity (media_entity)
         ├── pads[]
         └── /dev/v4l-subdev0

struct vb2_queue
├── type = V4L2_BUF_TYPE_VIDEO_CAPTURE
├── io_modes = VB2_MMAP | VB2_DMABUF
├── ops (vb2_ops)
│   ├── queue_setup
│   ├── buf_prepare
│   ├── buf_queue
│   ├── start_streaming
│   └── stop_streaming
├── bufs[] (array of vb2_buffer)
└── mem_ops (vb2_mem_ops: dma-contig / dma-sg / vmalloc)
```

---

## 19.3 DRM Data Structures

```
DRM Structure Map:

struct drm_device
├── dev (struct device)
├── driver (struct drm_driver)
│   ├── driver_features
│   ├── fops
│   └── DRM_GEM_CMA_DRIVER_OPS
├── mode_config (struct drm_mode_config)
│   ├── funcs (drm_mode_config_funcs)
│   │   ├── fb_create
│   │   ├── atomic_check
│   │   └── atomic_commit
│   ├── crtc_list (list of drm_crtc)
│   ├── plane_list (list of drm_plane)
│   ├── encoder_list
│   └── connector_list
│
├── struct drm_crtc
│   ├── primary (drm_plane)
│   ├── cursor (drm_plane)
│   ├── mode (drm_display_mode)
│   ├── state (drm_crtc_state) ← atomic state
│   └── funcs/helper_funcs
│       ├── atomic_enable
│       ├── atomic_disable
│       └── atomic_check
│
├── struct drm_plane
│   ├── type (PRIMARY / OVERLAY / CURSOR)
│   ├── fb (drm_framebuffer)
│   ├── state (drm_plane_state)
│   │   ├── fb, crtc
│   │   ├── src_x/y/w/h (source rect)
│   │   └── crtc_x/y/w/h (dest rect)
│   └── helper_funcs
│       ├── atomic_check
│       └── atomic_update
│
├── struct drm_encoder
│   ├── encoder_type (LVDS/HDMI/DSI)
│   └── possible_crtcs (bitmask)
│
└── struct drm_connector
    ├── connector_type
    ├── status (connected/disconnected)
    ├── modes (list of drm_display_mode)
    └── state (drm_connector_state)
```

---

## 19.4 ALSA/ASoC Data Structures

```
ASoC Structure Map:

struct snd_soc_card
├── name = "My-Board-Audio"
├── dai_link[] (struct snd_soc_dai_link)
│   ├── cpus[]     (COMP_CPU)      → platform driver
│   ├── codecs[]   (COMP_CODEC)    → codec driver
│   ├── platforms[] (COMP_PLATFORM) → DMA driver
│   └── dai_fmt (I2S, TDM, etc.)
├── dapm_widgets[]   (board-level DAPM)
├── dapm_routes[]    (board-level routes)
└── rtd[] (snd_soc_pcm_runtime — instantiated links)

struct snd_soc_component
├── name
├── driver (snd_soc_component_driver)
├── dai_list (list of snd_soc_dai)
├── controls[] (snd_kcontrol)
├── dapm_widgets[]
└── dapm_routes[]

struct snd_soc_dai
├── name = "i2s0"
├── playback (snd_soc_pcm_stream)
│   ├── channels_min/max
│   ├── rates
│   └── formats
├── capture (snd_soc_pcm_stream)
└── ops (snd_soc_dai_ops)
    ├── hw_params
    ├── set_fmt
    └── trigger

struct snd_pcm_substream
├── stream (PLAYBACK / CAPTURE)
├── runtime (snd_pcm_runtime)
│   ├── dma_area    (CPU address of ring buffer)
│   ├── dma_addr    (DMA address)
│   ├── dma_bytes   (buffer size)
│   ├── hw_ptr      (DMA position)
│   ├── appl_ptr    (app position)
│   ├── period_size
│   └── buffer_size
└── ops
```

---

## 19.5 Network Data Structures

```
Network Structure Map:

struct net_device
├── name = "eth0"
├── netdev_ops (net_device_ops)
│   ├── ndo_open, ndo_stop
│   ├── ndo_start_xmit
│   └── ndo_get_stats64
├── ethtool_ops
├── stats (rtnl_link_stats64)
│   ├── rx_packets, rx_bytes
│   └── tx_packets, tx_bytes
├── flags (IFF_UP, IFF_RUNNING)
├── mtu
├── dev_addr[6] (MAC address)
└── priv (private data via netdev_priv())

struct sk_buff
├── head, data, tail, end (buffer pointers)
├── len (data length)
├── protocol (ETH_P_IP, etc.)
├── dev (net_device)
├── cb[48] (control buffer — protocol scratch)
├── next/prev (list links)
├── sk (socket owner)
├── destructor
└── Transport/Network/MAC headers
    ├── transport_header → TCP/UDP header
    ├── network_header   → IP header
    └── mac_header       → Ethernet header
```

---

## 19.6 container_of and Embedding Patterns

```c
/* container_of — the kernel's most important macro */

#define container_of(ptr, type, member) ({               \
    void *__mptr = (void *)(ptr);                        \
    ((type *)(__mptr - offsetof(type, member)));          \
})

/* Example: recovering driver-specific struct from generic struct */

struct my_camera {
    struct video_device vdev;     /* embedded at some offset */
    struct v4l2_device v4l2_dev;
    void __iomem *regs;
    struct clk *clk;
    int frame_count;
};

/* In an ioctl handler, you receive struct file *filp */
static int my_ioctl(struct file *filp, unsigned int cmd, ...)
{
    struct video_device *vdev = video_devdata(filp);
    struct my_camera *cam = container_of(vdev, struct my_camera, vdev);
    /* Now you have the full driver structure */
}

/* Common to_* macros — all use container_of internally */
to_platform_device(dev)    → struct platform_device
to_i2c_client(dev)         → struct i2c_client
to_spi_device(dev)         → struct spi_device
netdev_priv(ndev)          → private data after net_device
video_get_drvdata(vdev)    → driver-specific data

/* Linked list traversal */
struct list_head {
    struct list_head *next, *prev;
};

list_for_each_entry(pos, head, member) {
    /* iterates over list, pos = container_of each link */
}
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| device.h | include/linux/device.h | Core device structures |
| videodev2.h | include/uapi/linux/videodev2.h | V4L2 userspace API |
| drm_device.h | include/drm/drm_device.h | DRM device structure |
| netdevice.h | include/linux/netdevice.h | net_device structure |
| soc.h | include/sound/soc.h | ASoC structures |
| kernel.h | include/linux/kernel.h | container_of macro |

---

## Interview Questions

**Q1: Explain the container_of macro and why it's essential.**
A: `container_of(ptr, type, member)` returns a pointer to the containing structure, given a pointer to one of its members. It calculates: `(type *)((char *)ptr - offsetof(type, member))`. This is essential because kernel frameworks use callbacks with generic types (`struct device *`), but drivers need their specific data (`struct my_driver *`). By embedding `struct device` inside `struct my_driver`, the driver can recover its private structure from any callback. It enables polymorphism in C without vtables.

**Q2: How are V4L2 data structures organized?**
A: There are four levels: (1) `v4l2_device` — one per hardware instance, groups all V4L2 components. (2) `video_device` — represents `/dev/videoN`, contains ioctl_ops and vb2_queue. (3) `v4l2_subdev` — represents one pipeline component (sensor, ISP, CSI) with its own ops. (4) `vb2_queue/vb2_buffer` — manages DMA buffers with queue_setup, buf_queue, start_streaming callbacks. Additionally, `media_device/media_entity/media_link` form the topology graph that maps how subdevs connect.

**Q3: How does the kernel track private driver data across frameworks?**
A: Two patterns: (1) **Embedding** — the driver structure contains the framework structure (`struct my_drv { struct platform_device pdev; ... }`). Recovery uses `container_of()`. (2) **driver_data pointer** — `dev_set_drvdata(dev, priv)` stores a void pointer in `struct device`. Retrieved with `dev_get_drvdata(dev)`. Most frameworks provide convenience helpers: `platform_set_drvdata()`, `i2c_set_clientdata()`, `video_set_drvdata()`, `netdev_priv()` (which returns the memory right after `struct net_device`).

---

## Summary

- Each framework builds on core `struct device` via embedding or pointer reference
- `container_of` is the key macro for recovering driver-specific types from generic types
- V4L2: v4l2_device → video_device + v4l2_subdev + vb2_queue
- DRM: drm_device → mode_config → crtc/plane/encoder/connector
- ASoC: snd_soc_card → dai_link → component → dai
- Network: net_device → sk_buff for packet management
- Linked lists (`list_head`) connect objects throughout the kernel

---

*Next: [Chapter 20 — Debugging Frameworks](Chapter_20_Debugging_Frameworks.md)*
