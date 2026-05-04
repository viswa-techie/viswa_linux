# Chapter 5: Graphics Framework (DRM/KMS)

## Learning Goals
- Understand DRM (Direct Rendering Manager) architecture
- Know KMS components: CRTC, encoder, connector, plane
- Grasp GEM buffer management and atomic modesetting
- Trace display pipeline from framebuffer to screen

---

## 5.1 DRM/KMS Architecture

```
DRM/KMS Architecture:

User Space:
  ┌──────────┐  ┌───────────┐  ┌───────────┐
  │ Weston   │  │ SurfFlngr │  │ modetest  │
  │ (Wayland)│  │ (Android) │  │ (debug)   │
  └─────┬────┘  └─────┬─────┘  └─────┬─────┘
        │              │              │
        └──────────────┼──────────────┘
                       │  /dev/dri/card0
Kernel:                │
  ┌────────────────────▼─────────────────────────────┐
  │              DRM Core                             │
  │                                                   │
  │  KMS (Kernel Mode Setting):                       │
  │  ┌─────────┐  ┌─────────┐  ┌──────────┐          │
  │  │  CRTC   │  │ Encoder │  │Connector │          │
  │  │ (pixel  │──│ (signal │──│ (physical│          │
  │  │  pipe)  │  │  format)│  │  port)   │          │
  │  └────┬────┘  └─────────┘  └──────────┘          │
  │       │                                           │
  │  ┌────▼────┐                                      │
  │  │  Plane  │  (overlays, cursor, primary)         │
  │  └─────────┘                                      │
  │                                                   │
  │  GEM (Graphics Execution Manager):                │
  │  ┌──────────────────────────────────────┐         │
  │  │ Buffer Objects (framebuffers)         │         │
  │  │ DMA-buf import/export                 │         │
  │  │ GEM handles, fencing, mmap            │         │
  │  └──────────────────────────────────────┘         │
  └──────────────────────────────────────────────────┘
                       │
Hardware:              ▼
  ┌────────┐  ┌────────┐  ┌──────────┐  ┌────────┐
  │ Memory │  │ Pixel  │  │ Display  │  │ HDMI/  │
  │ (VRAM/ │→ │ Pipe   │→ │ Encoder  │→ │ DSI/   │
  │  CMA)  │  │ (CRTC) │  │ (LVDS/   │  │ eDP    │
  │        │  │        │  │  HDMI)   │  │ Panel  │
  └────────┘  └────────┘  └──────────┘  └────────┘
```

---

## 5.2 KMS Objects in Detail

```
KMS Object Relationships:

                    ┌───────────────────────────────────────┐
                    │           Framebuffer (FB)             │
                    │  ● Pixel data (GEM buffer handle)     │
                    │  ● Width, height, format (XRGB8888)   │
                    │  ● Pitch (bytes per row)              │
                    └───────────────┬───────────────────────┘
                                    │ attached to
                    ┌───────────────▼───────────────────────┐
                    │             Plane                      │
                    │  ● Primary (main display content)      │
                    │  ● Overlay (HW compositing layer)      │
                    │  ● Cursor (mouse pointer)              │
                    │  Properties: src_x/y/w/h, crtc_x/y/w/h│
                    └───────────────┬───────────────────────┘
                                    │ feeds into
                    ┌───────────────▼───────────────────────┐
                    │             CRTC                       │
                    │  ● Pixel pipeline (scanout engine)     │
                    │  ● Timing generator (mode/resolution)  │
                    │  ● Gamma correction                    │
                    │  ● VBlank interrupt source              │
                    └───────────────┬───────────────────────┘
                                    │ outputs to
                    ┌───────────────▼───────────────────────┐
                    │            Encoder                     │
                    │  ● Signal format conversion            │
                    │  ● LVDS, HDMI, DSI, eDP, VGA           │
                    │  ● One CRTC → one or more encoders     │
                    └───────────────┬───────────────────────┘
                                    │ connects to
                    ┌───────────────▼───────────────────────┐
                    │           Connector                    │
                    │  ● Physical output port                │
                    │  ● Hotplug detection                   │
                    │  ● EDID reading (monitor capabilities) │
                    │  ● Connection status (connected/disc.) │
                    └───────────────────────────────────────┘

Typical automotive display: 2 CRTCs → 2 encoders → 2 connectors
  CRTC0 → DSI encoder → DSI connector → Instrument Cluster
  CRTC1 → LVDS encoder → LVDS connector → IVI Display
```

---

## 5.3 Atomic Modesetting

```
Atomic Modesetting — All-or-nothing display updates:

Legacy API (deprecated):
  drmModeSetCrtc(crtc, fb, ...)    → set display mode
  drmModeSetPlane(plane, fb, ...)  → update overlay
  Problem: Multiple calls → tearing, partial updates

Atomic API:
  ┌──────────────────────────────────────────────┐
  │  Atomic Commit Request                        │
  │                                               │
  │  CRTC 0: mode = 1920x1080@60Hz               │
  │  Plane 0: FB=5, src=(0,0,1920,1080)          │
  │           crtc=(0,0,1920,1080)                │
  │  Plane 1: FB=7, src=(0,0,320,240)            │
  │           crtc=(100,100,320,240)              │
  │  Connector 0: CRTC=0                          │
  │                                               │
  │  Flags: DRM_MODE_ATOMIC_ALLOW_MODESET         │
  │         DRM_MODE_PAGE_FLIP_EVENT               │
  └──────────────────────────┬───────────────────┘
                             │
                   ┌─────────▼──────────┐
                   │ DRM Atomic Check   │
                   │ ● Bandwidth OK?    │
                   │ ● Format supported?│
                   │ ● Planes overlap?  │
                   │ TEST_ONLY: return  │
                   │ result w/o commit  │
                   └─────────┬──────────┘
                             │ all OK
                   ┌─────────▼──────────┐
                   │ Atomic Commit      │
                   │ ● All changes at   │
                   │   next VBlank      │
                   │ ● No tearing       │
                   │ ● Rollback on fail │
                   └────────────────────┘
```

```c
/* Atomic commit from user space (libdrm) */
drmModeAtomicReq *req = drmModeAtomicAlloc();

/* Set plane properties */
drmModeAtomicAddProperty(req, plane_id, prop_fb_id, fb_id);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_id, crtc_id);
drmModeAtomicAddProperty(req, plane_id, prop_src_w, 1920 << 16);
drmModeAtomicAddProperty(req, plane_id, prop_src_h, 1080 << 16);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_w, 1920);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_h, 1080);

/* Commit atomically */
ret = drmModeAtomicCommit(fd, req,
        DRM_MODE_PAGE_FLIP_EVENT | DRM_MODE_ATOMIC_NONBLOCK,
        user_data);
```

---

## 5.4 GEM Buffer Objects

```c
/* GEM (Graphics Execution Manager) — buffer management */

/* Driver-side: Create a GEM object backed by CMA */
struct drm_gem_cma_object *obj;
obj = drm_gem_cma_create(drm_dev, size);
/* obj->vaddr = CPU virtual address */
/* obj->dma_addr = physical/DMA address for display controller */

/* Framebuffer creation (kernel side) */
static const struct drm_mode_config_funcs my_mode_config_funcs = {
    .fb_create = drm_gem_fb_create,  /* generic GEM FB creator */
    .atomic_check = drm_atomic_helper_check,
    .atomic_commit = drm_atomic_helper_commit,
};

/* DMA-buf export/import — zero-copy sharing */
/* GPU renders to a GEM buffer → exports as DMA-buf fd */
/* Display controller imports the fd → scanout directly */
/* No memory copy between GPU and display */
```

---

## 5.5 DRM Driver Structure

```c
/* Minimal DRM driver skeleton */

/* CRTC atomic functions */
static const struct drm_crtc_helper_funcs my_crtc_helper = {
    .mode_valid    = my_crtc_mode_valid,
    .atomic_check  = my_crtc_atomic_check,
    .atomic_enable = my_crtc_atomic_enable,   /* enable display */
    .atomic_disable = my_crtc_atomic_disable,
};

/* Plane atomic functions */
static const struct drm_plane_helper_funcs my_plane_helper = {
    .atomic_check  = my_plane_atomic_check,
    .atomic_update = my_plane_atomic_update,  /* program HW scanout */
};

static void my_plane_atomic_update(struct drm_plane *plane,
                                   struct drm_atomic_state *state)
{
    struct drm_plane_state *new_state =
        drm_atomic_get_new_plane_state(state, plane);
    struct drm_gem_cma_object *gem =
        drm_fb_cma_get_gem_obj(new_state->fb, 0);

    /* Program display controller with buffer address */
    writel(gem->dma_addr, regs + SCANOUT_ADDR);
    writel(new_state->crtc_w, regs + DISPLAY_WIDTH);
    writel(new_state->crtc_h, regs + DISPLAY_HEIGHT);
}

/* Encoder and connector */
static const struct drm_encoder_funcs my_encoder_funcs = {
    .destroy = drm_encoder_cleanup,
};

/* DRM driver definition */
DEFINE_DRM_GEM_CMA_FOPS(my_drm_fops);

static const struct drm_driver my_drm_driver = {
    .driver_features = DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC,
    .fops = &my_drm_fops,
    .name = "my-display",
    DRM_GEM_CMA_DRIVER_OPS,
};
```

---

## 5.6 Display Pipeline Debug

```bash
# DRM debug commands

# List DRM devices
$ ls /dev/dri/

# Dump current display state
$ cat /sys/kernel/debug/dri/0/state

# List connectors and modes
$ modetest -c     # connectors
$ modetest -p     # planes
$ modetest -e     # encoders

# Test a mode (display pattern)
$ modetest -s <connector_id>@<crtc_id>:1920x1080@60

# DRM debug messages
$ echo 0x1f > /sys/module/drm/parameters/debug

# Android: dumpsys
$ dumpsys SurfaceFlinger
$ dumpsys display
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| DRM core | drivers/gpu/drm/drm_drv.c | DRM driver registration |
| KMS | drivers/gpu/drm/drm_crtc.c | CRTC management |
| Atomic | drivers/gpu/drm/drm_atomic.c | Atomic modesetting |
| Atomic helpers | drivers/gpu/drm/drm_atomic_helper.c | Helper functions |
| GEM CMA | drivers/gpu/drm/drm_gem_cma_helper.c | CMA buffer objects |
| Connector | drivers/gpu/drm/drm_connector.c | Output port management |
| Panel | drivers/gpu/drm/panel/ | LCD panel drivers |
| Bridge | drivers/gpu/drm/bridge/ | Display bridge chips |

---

## Interview Questions

**Q1: Explain the KMS display pipeline objects.**
A: KMS has four main objects: (1) **Plane** — represents a layer of pixel data sourced from a framebuffer (primary=main content, overlay=HW composition, cursor=pointer). (2) **CRTC** — the pixel pipeline/scanout engine that reads planes and generates timed pixel output at a resolution and refresh rate. (3) **Encoder** — converts CRTC pixel output to a specific signal format (HDMI, DSI, LVDS, eDP). (4) **Connector** — represents the physical output port, handles hotplug detection and EDID reading. The chain is: Framebuffer → Plane → CRTC → Encoder → Connector → Display.

**Q2: What is atomic modesetting and why was it introduced?**
A: Atomic modesetting allows multiple display changes (mode, plane position, framebuffer) to be committed as a single atomic transaction. All changes take effect at the same VBlank, eliminating tearing and partial updates. It supports TEST_ONLY mode to validate a configuration without applying it. If any part of the commit fails validation, the entire commit is rejected — no partial state. Legacy API required multiple sequential ioctl calls that could result in inconsistent intermediate states.

**Q3: How does DMA-buf enable zero-copy display in Android?**
A: In Android, the GPU renders to a GEM buffer and exports it as a DMA-buf file descriptor. SurfaceFlinger passes this fd to HWComposer, which imports it into the DRM display driver. The display controller reads pixels directly from the GPU-rendered buffer via its DMA address — no CPU copy occurs. This is critical for performance: a 4K RGBA framebuffer is 33MB; copying it at 60fps would consume 2GB/s of memory bandwidth.

---

## Summary

- DRM/KMS is the Linux display framework replacing fbdev
- KMS objects: Plane → CRTC → Encoder → Connector form the display pipeline
- Atomic modesetting commits all display changes as one VBlank transaction
- GEM manages graphics buffer objects with DMA-buf for zero-copy sharing
- Planes support hardware compositing (primary + overlay + cursor layers)
- DRM drivers implement atomic_check/atomic_update for HW programming

---

*Next: [Chapter 6 — Input Subsystem](Chapter_06_Input_Subsystem.md)*
