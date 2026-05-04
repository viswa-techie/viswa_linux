# Chapter 22: Kernel I/O Subsystems

## Chapter Overview

The Linux kernel organizes drivers into I/O subsystems — frameworks that provide common infrastructure so individual drivers need only implement hardware-specific parts. This chapter covers the major subsystems: input, multimedia, graphics, and storage.

---

## 22.1 Linux I/O Architecture

```
┌──────────────────────────────────────────┐
│             User Space Application       │
├───────────┬───────────┬──────────────────┤
│  /dev/    │  sysfs    │ V4L2 / DRM APIs  │
├───────────┴───────────┴──────────────────┤
│        Virtual File System (VFS)         │
├───┬───────┬─────────┬──────────┬─────────┤
│   │ Input │  V4L2   │  DRM/KMS │  Block  │
│   │ Sub-  │  Frame- │  Frame-  │  Layer  │
│   │ system│  work   │  work    │ (blk-mq)│
├───┴───────┴─────────┴──────────┴─────────┤
│        Individual Hardware Drivers       │
├──────────────────────────────────────────┤
│              Hardware                    │
└──────────────────────────────────────────┘
```

---

## 22.2 Input Subsystem

Handles keyboards, mice, touchscreens, joysticks, accelerometers.

### Architecture

```
Hardware → IRQ → Input Driver → input_report_*() → Input Core → evdev → /dev/input/eventN → libinput → Application
```

### Key Structures

```c
#include <linux/input.h>

struct input_dev {
    const char *name;
    unsigned long evbit[BITS_TO_LONGS(EV_CNT)];   /* Supported event types */
    unsigned long keybit[BITS_TO_LONGS(KEY_CNT)];  /* Supported keys */
    unsigned long absbit[BITS_TO_LONGS(ABS_CNT)];  /* Absolute axes */
    /* ... */
};
```

### Minimal Input Driver

```c
static int my_input_probe(struct platform_device *pdev)
{
    struct input_dev *input;
    int ret;

    input = devm_input_allocate_device(&pdev->dev);
    if (!input)
        return -ENOMEM;

    input->name = "my-touchscreen";
    input->id.bustype = BUS_I2C;

    /* Report absolute X, Y, pressure */
    input_set_abs_params(input, ABS_X, 0, 1920, 0, 0);
    input_set_abs_params(input, ABS_Y, 0, 1080, 0, 0);
    input_set_abs_params(input, ABS_PRESSURE, 0, 255, 0, 0);

    /* Also report touch events */
    __set_bit(EV_ABS, input->evbit);
    __set_bit(EV_KEY, input->evbit);
    __set_bit(BTN_TOUCH, input->keybit);

    ret = input_register_device(input);
    if (ret)
        return ret;

    /* Request IRQ for touch events */
    return devm_request_threaded_irq(&pdev->dev, irq, NULL,
                my_touch_irq, IRQF_ONESHOT, "my-touch", input);
}

static irqreturn_t my_touch_irq(int irq, void *data)
{
    struct input_dev *input = data;
    int x, y, pressed;

    /* Read from hardware ... */

    if (pressed) {
        input_report_abs(input, ABS_X, x);
        input_report_abs(input, ABS_Y, y);
        input_report_abs(input, ABS_PRESSURE, 128);
        input_report_key(input, BTN_TOUCH, 1);
    } else {
        input_report_key(input, BTN_TOUCH, 0);
        input_report_abs(input, ABS_PRESSURE, 0);
    }
    input_sync(input);   /* Report complete event packet */

    return IRQ_HANDLED;
}
```

### Event Types

| Event | Code | Description |
|-------|------|-------------|
| EV_KEY | 0x01 | Key press/release |
| EV_REL | 0x02 | Relative movement (mouse) |
| EV_ABS | 0x03 | Absolute position (touchscreen) |
| EV_MSC | 0x04 | Miscellaneous |
| EV_SW | 0x05 | Switch (lid, headphone) |
| EV_FF | 0x15 | Force feedback |

---

## 22.3 V4L2 (Video4Linux2) — Multimedia Subsystem

Handles cameras, video encoders/decoders, TV tuners.

### Architecture

```
Application (GStreamer, ffmpeg)
        │ ioctl: VIDIOC_QBUF, VIDIOC_DQBUF
        ▼
   V4L2 Core (/dev/videoN)
        │
   videobuf2 (vb2) buffer management
        │
   Hardware-specific driver (capture/m2m/output)
        │
   DMA engine / hardware codec
```

### Key APIs

```c
#include <media/v4l2-device.h>
#include <media/v4l2-ioctl.h>
#include <media/videobuf2-dma-contig.h>

static const struct v4l2_ioctl_ops my_ioctl_ops = {
    .vidioc_querycap         = my_querycap,
    .vidioc_enum_fmt_vid_cap = my_enum_fmt,
    .vidioc_g_fmt_vid_cap    = my_g_fmt,
    .vidioc_s_fmt_vid_cap    = my_s_fmt,
    .vidioc_reqbufs          = vb2_ioctl_reqbufs,
    .vidioc_qbuf             = vb2_ioctl_qbuf,
    .vidioc_dqbuf            = vb2_ioctl_dqbuf,
    .vidioc_streamon         = vb2_ioctl_streamon,
    .vidioc_streamoff        = vb2_ioctl_streamoff,
};
```

### Buffer Management (vb2)

```c
static const struct vb2_ops my_vb2_ops = {
    .queue_setup     = my_queue_setup,     /* Allocate buffers */
    .buf_prepare     = my_buf_prepare,     /* Validate before queueing */
    .buf_queue       = my_buf_queue,       /* Queue buffer to hardware */
    .start_streaming = my_start_streaming, /* Start DMA */
    .stop_streaming  = my_stop_streaming,  /* Stop DMA */
};
```

---

## 22.4 DRM/KMS — Graphics Subsystem

Handles display controllers, GPUs, framebuffers.

### Architecture

```
Wayland/Xorg
    │
    ▼ (DRM ioctls)
DRM Core (/dev/dri/cardN)
    │
    ├── KMS (Kernel Mode Setting)
    │   ├── CRTC → Scanout engine
    │   ├── Encoder → Signal conversion (HDMI, LVDS)
    │   ├── Connector → Physical port
    │   └── Plane → Overlay / framebuffer
    │
    └── GEM (Graphics Execution Manager)
        └── Buffer allocation/mapping
```

### Key Structures

```c
#include <drm/drm_drv.h>
#include <drm/drm_crtc.h>

static const struct drm_driver my_drm_driver = {
    .driver_features = DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC,
    .name            = "my-display",
    .fops            = &my_drm_fops,
    /* GEM helpers */
    .dumb_create     = drm_gem_dma_dumb_create,
};

/* Minimal CRTC: enable/disable display pipeline */
static const struct drm_crtc_helper_funcs my_crtc_helper = {
    .mode_set_nofb  = my_crtc_mode_set,
    .atomic_enable  = my_crtc_enable,
    .atomic_disable = my_crtc_disable,
};
```

---

## 22.5 Storage Subsystem

See Chapter 9 for blk-mq. Here we cover the higher-level layers.

```
Filesystem (ext4, f2fs, btrfs)
    │
    ▼
VFS (struct bio)
    │
    ▼
Block Layer (blk-mq)
    │
    ├── I/O schedulers (mq-deadline, bfq, kyber, none)
    │
    ▼
Block driver (NVMe, SCSI, MMC, virtio-blk)
    │
    ▼
Hardware (SSD, eMMC, UFS)
```

### SCSI Subsystem (Widely Used)

```c
#include <scsi/scsi_host.h>
#include <scsi/scsi_cmnd.h>

static struct scsi_host_template my_sht = {
    .module         = THIS_MODULE,
    .name           = "my-storage",
    .queuecommand   = my_queuecommand,    /* Submit SCSI command */
    .eh_abort_handler = my_abort,
    .can_queue      = 256,
    .sg_tablesize   = SG_ALL,
};
```

---

## 22.6 Subsystem Comparison

| Subsystem | Device node | Framework | Header |
|-----------|-------------|-----------|--------|
| Input | /dev/input/eventN | input_dev | linux/input.h |
| V4L2 | /dev/videoN | v4l2_device, vb2 | media/v4l2-device.h |
| DRM/KMS | /dev/dri/cardN | drm_device | drm/drm_drv.h |
| Block | /dev/sdX, /dev/nvmeN | gendisk, blk-mq | linux/blk-mq.h |
| Sound | /dev/snd/* | snd_card | sound/core.h |
| GPIO | /dev/gpiochipN | gpio_chip | linux/gpio/driver.h |
| IIO | /dev/iio:deviceN | iio_dev | linux/iio/iio.h |

---

## Kernel Source References

| File | Content |
|------|---------|
| drivers/input/input.c | Input core |
| drivers/media/v4l2-core/ | V4L2 framework |
| drivers/gpu/drm/drm_drv.c | DRM core |
| block/blk-mq.c | Multi-queue block layer |
| drivers/iio/industrialio-core.c | IIO framework |

---

## Interview Questions

**Q1: What are the advantages of using a subsystem framework vs writing raw char driver?**
A: Frameworks provide: standard userspace API (evdev, V4L2, DRM), common buffer management, power management integration, sysfs attributes, and existing userspace tooling (libinput, GStreamer, Wayland). Writing a raw char driver duplicates this infrastructure.

**Q2: How does input_sync() work?**
A: `input_sync()` sends an `EV_SYN/SYN_REPORT` event to mark the end of an atomic input event. Userspace reads complete event packets between `SYN_REPORT` markers, ensuring consistent multi-axis data (e.g., X and Y of a touch are always paired).

**Q3: What is the difference between DRM GEM and dma-buf?**
A: GEM (Graphics Execution Manager) manages GPU buffer objects within one DRM driver. dma-buf is a kernel framework for sharing buffers between different drivers (e.g., camera DMA → GPU rendering). A GEM object can be exported as a dma-buf fd.

---

*Next: [Chapter 23 — Important Kernel Data Structures](Chapter_23_Data_Structures.md)*
