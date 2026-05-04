# Chapter 24: References and Resources

## Learning Goals
- Know the essential books, documentation, and source references
- Find kernel documentation for each framework
- Access community resources and mailing lists
- Build a continuous learning path for Linux device driver development

---

## 24.1 Essential Books

```
Recommended Reading:

Foundation:
├── "Linux Device Drivers, 3rd Ed" — Corbet, Rubini, Kroah-Hartman
│   └── Core concepts: char/block devices, memory, interrupts
├── "Linux Kernel Development, 3rd Ed" — Robert Love
│   └── Process scheduling, memory management, VFS
├── "Understanding the Linux Kernel, 3rd Ed" — Bovet, Cesati
│   └── Deep internals: VM, page cache, process management
└── "Essential Linux Device Drivers" — Sreekrishnan Venkateswaran
    └── Practical driver examples across subsystems

Framework-Specific:
├── "Linux Driver Development with Raspberry Pi" — Madieu
│   └── Hands-on: I2C, SPI, GPIO, DMA, IIO, input
├── "Mastering Linux Device Driver Development" — Madieu
│   └── V4L2, DRM, ALSA, clock, regulator, thermal
└── "Linux Kernel Programming" — Billimoria
    └── Modern kernel module development

Embedded / Automotive:
├── "Embedded Linux Primer" — Hallinan
├── "Building Embedded Linux Systems" — Yaghmour
└── "Automotive Ethernet" — Kirsten Matheus
```

---

## 24.2 Kernel Documentation (In-Tree)

```
Kernel Documentation Tree (Documentation/):

Framework Docs:
├── driver-api/
│   ├── media/         ← V4L2, media controller, DMA-buf
│   ├── gpu/           ← DRM/KMS, GEM, atomic
│   ├── sound/         ← ALSA, ASoC, DAPM
│   ├── input.rst      ← Input subsystem
│   ├── i2c/           ← I2C framework
│   ├── spi/           ← SPI framework
│   ├── gpio/          ← GPIO subsystem
│   ├── dmaengine/     ← DMA engine API
│   ├── clk.rst        ← Clock framework
│   ├── regulator.rst  ← Regulator framework
│   └── thermal/       ← Thermal management
│
├── core-api/
│   ├── dma-api.rst          ← DMA mapping API
│   ├── device_link.rst      ← Device links
│   └── kobject.rst          ← kobject/sysfs
│
├── devicetree/
│   └── bindings/      ← All DT binding schemas
│       ├── media/
│       ├── display/
│       ├── sound/
│       ├── input/
│       ├── i2c/
│       ├── spi/
│       ├── gpio/
│       ├── dma/
│       ├── clock/
│       ├── regulator/
│       └── thermal/
│
├── power/
│   ├── runtime_pm.rst       ← Runtime PM guide
│   ├── suspend-and-resume.rst
│   └── regulator/
│
└── networking/
    └── can.rst               ← CAN / SocketCAN

Access:
  Online: https://docs.kernel.org/
  In-tree: make htmldocs → Documentation/output/
```

---

## 24.3 Kernel Source — Key Files Per Framework

```
Quick Reference: Where to Read Source

V4L2:
  drivers/media/v4l2-core/v4l2-dev.c        (video device)
  drivers/media/v4l2-core/v4l2-ioctl.c       (ioctl dispatch)
  drivers/media/common/videobuf2/            (buffer management)
  include/media/v4l2-device.h                (core structures)

DRM/KMS:
  drivers/gpu/drm/drm_drv.c                  (DRM device)
  drivers/gpu/drm/drm_atomic.c               (atomic commit)
  drivers/gpu/drm/drm_atomic_helper.c        (helper functions)
  include/drm/drm_crtc.h                     (KMS objects)

ALSA/ASoC:
  sound/soc/soc-core.c                       (ASoC card/DAI)
  sound/soc/soc-pcm.c                        (PCM operations)
  sound/soc/soc-dapm.c                       (DAPM power)
  include/sound/soc.h                        (ASoC structures)

Input:
  drivers/input/input.c                      (core dispatch)
  drivers/input/evdev.c                      (evdev handler)
  include/linux/input.h                      (input_dev)

Network:
  net/core/dev.c                             (net_device)
  net/core/skbuff.c                          (sk_buff)
  include/linux/netdevice.h                  (structures)

I2C:
  drivers/i2c/i2c-core-base.c               (core)
  include/linux/i2c.h                        (structures)

SPI:
  drivers/spi/spi.c                          (core)
  include/linux/spi/spi.h                    (structures)

GPIO:
  drivers/gpio/gpiolib.c                     (core)
  include/linux/gpio/driver.h               (gpio_chip)
  include/linux/gpio/consumer.h             (gpiod API)

DMA:
  drivers/dma/dmaengine.c                   (core)
  include/linux/dmaengine.h                 (consumer API)

Clock:
  drivers/clk/clk.c                         (CCF core)
  include/linux/clk.h                       (consumer API)

Regulator:
  drivers/regulator/core.c                  (framework)
  include/linux/regulator/consumer.h        (consumer API)

PM:
  drivers/base/power/runtime.c              (Runtime PM)
  include/linux/pm_runtime.h                (API)

Thermal:
  drivers/thermal/thermal_core.c            (core)
  include/linux/thermal.h                   (API)
```

---

## 24.4 Tools and Utilities

```
Essential Development Tools:

Kernel Build:
  make menuconfig           # configure kernel
  make -j$(nproc)           # build kernel
  make dtbs                 # compile device trees
  make modules              # build modules
  make M=drivers/my/ modules # build specific module

Debug Tools:
  dmesg                     # kernel messages
  ftrace                    # function tracing
  trace-cmd                 # ftrace front-end
  perf                      # performance profiling
  strace                    # system call tracing
  devmem2                   # direct register access

Framework-Specific Tools:
  v4l2-ctl / v4l2-compliance  ← V4L2
  media-ctl                    ← Media controller
  modetest / drm_info          ← DRM/KMS
  aplay / arecord / amixer     ← ALSA
  evtest / libgpiod            ← Input / GPIO
  i2cdetect / i2cget / i2cdump ← I2C
  lsusb / usbmon               ← USB
  can-utils (cansend/candump)   ← CAN
  ethtool / ip / ss             ← Network

Code Quality:
  checkpatch.pl             # coding style check
  sparse                    # static analysis
  coccinelle                # semantic patches
  W=1 / W=2                 # extra compiler warnings
  KASAN / UBSAN             # runtime sanitizers
```

---

## 24.5 Community Resources

```
Mailing Lists:
  LKML (linux-kernel@vger.kernel.org)        — General kernel
  linux-media@vger.kernel.org                — V4L2/media
  dri-devel@lists.freedesktop.org            — DRM/KMS
  alsa-devel@alsa-project.org                — ALSA/ASoC
  linux-input@vger.kernel.org                — Input
  linux-i2c@vger.kernel.org                  — I2C
  linux-spi@vger.kernel.org                  — SPI
  linux-gpio@vger.kernel.org                 — GPIO
  dmaengine@vger.kernel.org                  — DMA engine
  linux-clk@vger.kernel.org                  — Clock
  linux-pm@vger.kernel.org                   — Power management
  linux-can@vger.kernel.org                  — CAN / SocketCAN

Code Browsing:
  elixir.bootlin.com                         — Cross-referenced source
  lxr.linux.no                               — LXR
  git.kernel.org                             — Official git

Documentation:
  docs.kernel.org                            — Official docs
  kernelnewbies.org                          — Beginner resources
  lwn.net                                    — Linux Weekly News
  elinux.org                                 — Embedded Linux wiki
```

---

## 24.6 Learning Path by Role

```
Learning Paths:

Embedded Linux Developer:
  Ch 1-2 (Foundations) → Ch 9-11 (I2C/SPI/GPIO) →
  Ch 12 (DMA) → Ch 13-14 (Clock/Regulator) →
  Ch 15 (PM) → Ch 17 (DT) → Ch 23 (Embedded Patterns)

Camera/Display Developer:
  Ch 1-2 (Foundations) → Ch 3 (V4L2) → Ch 5 (DRM) →
  Ch 12 (DMA) → Ch 18 (Interactions) →
  Ch 21 (Flow Diagrams) → Ch 22 (Automotive)

Audio Developer:
  Ch 1-2 (Foundations) → Ch 4 (ALSA/ASoC) →
  Ch 9 (I2C) → Ch 12 (DMA) → Ch 13-14 (Clock/Regulator) →
  Ch 22 (Automotive Audio)

Automotive Developer:
  Ch 1-2 (Foundations) → Ch 7 (Network/CAN) →
  Ch 15 (PM) → Ch 22 (Automotive) →
  Ch 3 (V4L2) → Ch 5 (DRM) → Ch 4 (ALSA)

Debug/Integration Engineer:
  Ch 1-2 (Foundations) → Ch 17 (DT) → Ch 18 (Interactions) →
  Ch 20 (Debugging) → Ch 21 (Flow Diagrams) →
  Ch 19 (Data Structures)

Interview Preparation:
  Ch 1-2 → Ch 25 (Interview) → Ch 19 (Data Structures) →
  Ch 21 (Flow Diagrams) → All framework chapters for depth
```

---

## Summary

- Linux kernel documentation is the primary reference: docs.kernel.org
- Each framework has dedicated kernel source, DT bindings, and mailing list
- In-tree docs (Documentation/) contain binding schemas and API guides
- elixir.bootlin.com provides cross-referenced kernel source browsing
- Framework-specific tools are essential: v4l2-ctl, modetest, amixer, i2cdetect
- Follow learning paths by role for efficient knowledge building

---

*Next: [Chapter 25 — Interview Preparation](Chapter_25_Interview_Preparation.md)*
