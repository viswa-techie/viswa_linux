# Chapter 3: Linux Media Framework (V4L2)

## Learning Goals
- Understand V4L2 architecture and media controller
- Know video device nodes, buffer management, streaming pipeline
- Grasp subdev model and media entity linking
- Write or analyze a V4L2 driver

---

## 3.1 V4L2 Architecture

```
V4L2 Architecture:

User Space:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ GStreamer │  │ FFmpeg   │  │ v4l2-ctl │
  └─────┬────┘  └─────┬────┘  └─────┬────┘
        │              │              │
        └──────────────┼──────────────┘
                       │ ioctl (/dev/video0)
Kernel:                │
  ┌────────────────────▼─────────────────────────────┐
  │              V4L2 Core                            │
  │  ┌──────────────────────────────────────────┐    │
  │  │  video_device                             │    │
  │  │  ├── v4l2_file_operations (open/close)    │    │
  │  │  ├── v4l2_ioctl_ops (set_fmt, streamon)   │    │
  │  │  └── vb2_queue (buffer management)        │    │
  │  └──────────────────────────────────────────┘    │
  │  ┌──────────────────────────────────────────┐    │
  │  │  Media Controller                         │    │
  │  │  ├── media_device (top-level)             │    │
  │  │  ├── media_entity (each component)        │    │
  │  │  └── media_link (connections)             │    │
  │  └──────────────────────────────────────────┘    │
  │  ┌──────────────────────────────────────────┐    │
  │  │  V4L2 Subdevice (v4l2_subdev)             │    │
  │  │  ├── Sensor driver (IMX219, OV5640)       │    │
  │  │  ├── ISP driver (image processing)        │    │
  │  │  └── CSI receiver driver                  │    │
  │  └──────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘
```

---

## 3.2 Media Controller Framework

```
Media Controller — Complex Camera Pipelines:

Real camera system:
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Sensor  │────►│  CSI-2   │────►│   ISP    │────►│  Video   │
│  IMX219  │MIPI │ Receiver │     │ (process)│     │  Capture │
│          │     │          │     │ denoise  │     │ /dev/    │
│  subdev  │     │  subdev  │     │ scale    │     │ video0   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
   entity 0        entity 1        entity 2         entity 3

Media links (configured via /dev/media0):
  entity0:pad0 → entity1:pad0     (sensor → CSI)
  entity1:pad1 → entity2:pad0     (CSI → ISP)
  entity2:pad1 → entity3:pad0     (ISP → capture)

Each entity:
  ├── Has pads (input/output ports)
  ├── Has links (connections between pads)
  ├── Has a function (sensor, processor, IO)
  └── Is controlled via its own /dev/v4l-subdevN
```

```c
/* Register a media device */
struct media_device *mdev = devm_kzalloc(dev, sizeof(*mdev), GFP_KERNEL);
media_device_init(mdev);
strlcpy(mdev->model, "My Camera", sizeof(mdev->model));
mdev->dev = &pdev->dev;
ret = media_device_register(mdev);

/* Register a media entity */
my_entity.function = MEDIA_ENT_F_CAM_SENSOR;
my_pads[0].flags = MEDIA_PAD_FL_SOURCE;
ret = media_entity_pads_init(&my_entity, 1, my_pads);
ret = media_device_register_entity(mdev, &my_entity);

/* Create link between entities */
ret = media_create_pad_link(&sensor_entity, 0,   /* source pad */
                            &csi_entity, 0,       /* sink pad */
                            MEDIA_LNK_FL_ENABLED);
```

---

## 3.3 Video Device Nodes

```c
/* Registering a V4L2 video device */

/* 1. Allocate and initialize v4l2_device (per hardware instance) */
struct v4l2_device v4l2_dev;
ret = v4l2_device_register(&pdev->dev, &v4l2_dev);

/* 2. Create video_device */
struct video_device *vdev = video_device_alloc();
vdev->fops = &my_v4l2_fops;        /* file operations */
vdev->ioctl_ops = &my_ioctl_ops;    /* ioctl handlers */
vdev->v4l2_dev = &v4l2_dev;
vdev->queue = &my_vb2_queue;        /* buffer queue */
vdev->release = video_device_release;
vdev->device_caps = V4L2_CAP_VIDEO_CAPTURE |
                    V4L2_CAP_STREAMING;

/* 3. Register — creates /dev/videoN */
ret = video_register_device(vdev, VFL_TYPE_VIDEO, -1);

/* ioctl operations */
static const struct v4l2_ioctl_ops my_ioctl_ops = {
    .vidioc_querycap      = my_querycap,
    .vidioc_enum_fmt_vid_cap = my_enum_fmt,
    .vidioc_s_fmt_vid_cap = my_s_fmt,
    .vidioc_g_fmt_vid_cap = my_g_fmt,
    .vidioc_reqbufs       = vb2_ioctl_reqbufs,
    .vidioc_querybuf      = vb2_ioctl_querybuf,
    .vidioc_qbuf          = vb2_ioctl_qbuf,
    .vidioc_dqbuf         = vb2_ioctl_dqbuf,
    .vidioc_streamon      = vb2_ioctl_streamon,
    .vidioc_streamoff     = vb2_ioctl_streamoff,
};
```

---

## 3.4 Buffer Management (videobuf2)

```
videobuf2 (vb2) — Buffer Management:

User Space:                          Kernel (vb2):
┌────────────────────┐              ┌────────────────────┐
│ REQBUFS(count=4)   │─────────────►│ Allocate 4 buffers │
│                    │              │ (DMA memory)       │
├────────────────────┤              ├────────────────────┤
│ QBUF(index=0)      │─────────────►│ Queue buf 0        │
│ QBUF(index=1)      │─────────────►│ Queue buf 1        │
│ QBUF(index=2)      │─────────────►│ Queue buf 2        │
├────────────────────┤              ├────────────────────┤
│ STREAMON           │─────────────►│ Start HW streaming │
│                    │              │ DMA fills buf 0    │
│                    │              │ ISR: buf 0 DONE    │
├────────────────────┤              ├────────────────────┤
│ DQBUF → buf 0     │◄─────────────│ Return filled buf  │
│ (process frame)    │              │                    │
│ QBUF(index=0)      │─────────────►│ Re-queue buf 0     │
├────────────────────┤              ├────────────────────┤
│ DQBUF → buf 1     │◄─────────────│ Return next frame  │
│ ...                │              │ ...                │
└────────────────────┘              └────────────────────┘

Memory types:
  V4L2_MEMORY_MMAP    → Kernel allocates, user mmaps
  V4L2_MEMORY_USERPTR → User allocates, passes pointer
  V4L2_MEMORY_DMABUF  → Share DMA-buf between devices
```

```c
/* vb2 queue setup in driver */
static const struct vb2_ops my_vb2_ops = {
    .queue_setup     = my_queue_setup,    /* allocate buffers */
    .buf_prepare     = my_buf_prepare,    /* validate buffer */
    .buf_queue       = my_buf_queue,      /* submit to HW */
    .start_streaming = my_start_streaming,/* start DMA */
    .stop_streaming  = my_stop_streaming, /* stop DMA */
};

static int my_queue_setup(struct vb2_queue *q,
                          unsigned int *nbuffers,
                          unsigned int *nplanes,
                          unsigned int sizes[],
                          struct device *alloc_devs[])
{
    *nplanes = 1;
    sizes[0] = width * height * 2;  /* YUYV: 2 bytes/pixel */
    return 0;
}

static void my_buf_queue(struct vb2_buffer *vb)
{
    /* Get DMA address of this buffer */
    dma_addr_t addr = vb2_dma_contig_plane_dma_addr(vb, 0);
    /* Program DMA to write camera data to this address */
    writel(addr, regs + DMA_ADDR_REG);
    writel(DMA_START, regs + DMA_CTRL_REG);
}
```

---

## 3.5 V4L2 Subdevice Model

```c
/* V4L2 subdev — for individual pipeline components */

/* Sensor driver (e.g., IMX219) */
static const struct v4l2_subdev_video_ops imx219_video_ops = {
    .s_stream = imx219_set_stream,  /* start/stop streaming */
};

static const struct v4l2_subdev_pad_ops imx219_pad_ops = {
    .enum_mbus_code  = imx219_enum_mbus_code,
    .set_fmt         = imx219_set_fmt,
    .get_fmt         = imx219_get_fmt,
};

static const struct v4l2_subdev_ops imx219_subdev_ops = {
    .video = &imx219_video_ops,
    .pad   = &imx219_pad_ops,
};

static int imx219_probe(struct i2c_client *client)
{
    struct v4l2_subdev *sd;

    sd = devm_kzalloc(&client->dev, sizeof(*sd), GFP_KERNEL);
    v4l2_i2c_subdev_init(sd, client, &imx219_subdev_ops);

    sd->flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;
    sd->entity.function = MEDIA_ENT_F_CAM_SENSOR;

    /* Initialize pad */
    pads[0].flags = MEDIA_PAD_FL_SOURCE;
    media_entity_pads_init(&sd->entity, 1, pads);

    return v4l2_async_register_subdev(sd);
}
```

---

## 3.6 User-Space Tools

```bash
# V4L2 user-space commands

# List video devices
$ v4l2-ctl --list-devices

# Query capabilities
$ v4l2-ctl -d /dev/video0 --all

# List supported formats
$ v4l2-ctl -d /dev/video0 --list-formats-ext

# Set format
$ v4l2-ctl -d /dev/video0 --set-fmt-video=width=1920,height=1080,pixelformat=YUYV

# Capture a frame
$ v4l2-ctl -d /dev/video0 --stream-mmap --stream-count=1 --stream-to=frame.raw

# Media controller
$ media-ctl -d /dev/media0 --print-topology
$ media-ctl -d /dev/media0 -l "'imx219 0-0010':0->'csi2':0[1]"

# V4L2 compliance test
$ v4l2-compliance -d /dev/video0
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| V4L2 core | drivers/media/v4l2-core/ | Core V4L2 framework |
| video_device | drivers/media/v4l2-core/v4l2-dev.c | Video device registration |
| v4l2_ioctl | drivers/media/v4l2-core/v4l2-ioctl.c | ioctl dispatch |
| videobuf2 | drivers/media/common/videobuf2/ | Buffer management |
| media controller | drivers/media/mc/ | Media entity management |
| v4l2_subdev | drivers/media/v4l2-core/v4l2-subdev.c | Subdevice framework |

---

## Interview Questions

**Q1: Explain the V4L2 streaming pipeline from user space to hardware.**
A: (1) App opens `/dev/video0` and sets format with `VIDIOC_S_FMT`. (2) App requests buffers with `VIDIOC_REQBUFS` — vb2 allocates DMA-capable memory. (3) App queues buffers with `VIDIOC_QBUF` — vb2 submits DMA addresses to the driver. (4) App calls `VIDIOC_STREAMON` — driver starts DMA and camera sensor streaming. (5) Hardware fills buffers via DMA; ISR marks buffers DONE. (6) App dequeues filled buffers with `VIDIOC_DQBUF`, processes the frame, and re-queues.

**Q2: What is the media controller and why is it needed?**
A: The media controller manages complex camera pipelines with multiple processing stages (sensor → CSI → ISP → capture). Each stage is a `media_entity` with pads (ports). `media_link` connects pads between entities. User space uses `/dev/media0` to discover the pipeline topology and configure links. Without it, a simple `/dev/video0` can't represent which sensor feeds which ISP configuration. It enables runtime pipeline reconfiguration for different use cases.

**Q3: What is the difference between V4L2_MEMORY_MMAP and V4L2_MEMORY_DMABUF?**
A: MMAP: Kernel allocates buffers, user space mmaps them into its address space. Simple, one-device usage. DMABUF: Buffers are represented as file descriptors (dma-buf) that can be shared between devices — e.g., camera captures to a buffer, GPU reads the same buffer for rendering. Zero-copy pipeline. DMABUF is essential for multi-device pipelines (camera → GPU → display) in automotive and mobile.

---

## Summary

- V4L2 is the Linux framework for video capture, output, and processing
- The media controller manages complex pipelines with entities, pads, and links
- videobuf2 handles buffer allocation, queueing, and DMA management
- V4L2 subdevices represent individual pipeline components (sensor, ISP, CSI)
- Streaming uses REQBUFS → QBUF → STREAMON → DQBUF cycle
- DMA-buf enables zero-copy buffer sharing between frameworks (V4L2 → DRM)

---

*Next: [Chapter 4 — Audio Framework (ALSA)](Chapter_04_ALSA_Audio.md)*
