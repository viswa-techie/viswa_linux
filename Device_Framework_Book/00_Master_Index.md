# Linux Device Frameworks — Complete Guide

## Master Index

---

## Book Overview

This book provides a **comprehensive A-to-Z guide** to the major Linux kernel device frameworks — the standardized subsystems that connect user-space applications to hardware through well-defined APIs. Covers **Video4Linux2, ALSA, DRM/KMS, Input, I2C, SPI, GPIO, USB, Networking, DMA, Clock, Regulator, Power Management, Thermal**, and their interactions within real embedded/automotive systems.

**Target audience**: Embedded Linux engineers, BSP developers, driver writers, automotive software engineers.
**Kernel versions**: 5.x — 6.x (concepts apply broadly).
**Architectures**: ARM64, x86-64 (framework layer is architecture-independent).

---

## Part I — Foundations (Chapters 1–2)

| # | Chapter | Key Topics |
|---|---------|-----------|
| 1 | [Foundations of Linux Device Frameworks](Chapter_01_Foundations.md) | What is a framework, abstraction, framework vs driver |
| 2 | [Linux Device Model Overview](Chapter_02_Device_Model.md) | struct device/driver/bus, sysfs, kobject, device hierarchy |

## Part II — Media & Display Frameworks (Chapters 3–5)

| # | Chapter | Key Topics |
|---|---------|-----------|
| 3 | [Linux Media Framework (V4L2)](Chapter_03_V4L2_Media.md) | V4L2 architecture, media controller, video nodes, streaming |
| 4 | [Audio Framework (ALSA)](Chapter_04_ALSA_Audio.md) | ALSA core, PCM streaming, mixer controls, ASoC |
| 5 | [Graphics / Display Framework (DRM/KMS)](Chapter_05_DRM_Graphics.md) | DRM architecture, KMS, framebuffers, planes, CRTCs, GPU |

## Part III — Input, Networking & USB (Chapters 6–8)

| # | Chapter | Key Topics |
|---|---------|-----------|
| 6 | [Input Device Framework](Chapter_06_Input_Subsystem.md) | Input events, evdev, keyboards, touchscreens, event reporting |
| 7 | [Networking Framework](Chapter_07_Networking.md) | net_device, sk_buff, NAPI, packet pipeline, ethtool |
| 8 | [USB Framework](Chapter_08_USB.md) | USB core, enumeration, host controllers, gadget, URBs |

## Part IV — Bus Frameworks (Chapters 9–11)

| # | Chapter | Key Topics |
|---|---------|-----------|
| 9 | [I2C Framework](Chapter_09_I2C.md) | I2C architecture, adapter/client drivers, SMBus, DT |
| 10 | [SPI Framework](Chapter_10_SPI.md) | SPI architecture, controller/device drivers, transfers |
| 11 | [GPIO Framework](Chapter_11_GPIO.md) | gpiochip, GPIO descriptors, interrupts, pinctrl |

## Part V — DMA, Clock, Regulator & Power (Chapters 12–16)

| # | Chapter | Key Topics |
|---|---------|-----------|
| 12 | [DMA Framework](Chapter_12_DMA.md) | DMA engine, mapping APIs, scatter-gather, coherent DMA |
| 13 | [Clock Framework](Chapter_13_Clock.md) | clk API, clock tree, gating, mux, dividers, DT clocks |
| 14 | [Regulator Framework](Chapter_14_Regulator.md) | Voltage/current regulators, constraints, coupling |
| 15 | [Power Management Framework](Chapter_15_Power_Management.md) | Runtime PM, suspend/resume, PM domains, wakeup sources |
| 16 | [Thermal Framework](Chapter_16_Thermal.md) | Thermal zones, cooling devices, governors, throttling |

## Part VI — Integration & Advanced (Chapters 17–21)

| # | Chapter | Key Topics |
|---|---------|-----------|
| 17 | [Device Tree Integration](Chapter_17_Device_Tree.md) | DT bindings for frameworks, parsing, overlays |
| 18 | [Framework Interaction Architecture](Chapter_18_Interaction_Architecture.md) | Cross-framework data flow, pipeline composition |
| 19 | [Framework Data Structures](Chapter_19_Data_Structures.md) | Core structs, relationships, lifecycle |
| 20 | [Framework Debugging](Chapter_20_Debugging.md) | dmesg, debugfs, tracing, sysfs inspection, tools |
| 21 | [Framework Flow Diagrams](Chapter_21_Flow_Diagrams.md) | Video/audio/network/USB pipeline diagrams |

## Part VII — Embedded & Interview (Chapters 22–25)

| # | Chapter | Key Topics |
|---|---------|-----------|
| 22 | [Embedded System Framework Architecture](Chapter_22_Embedded_Architecture.md) | SoC framework stack, real hardware integration |
| 23 | [Frameworks in Embedded Platforms](Chapter_23_Embedded_Platforms.md) | Automotive, mobile, industrial, IoT use cases |
| 24 | [Framework Documentation & References](Chapter_24_References.md) | Books, kernel docs, online resources, tools |
| 25 | [Interview Preparation](Chapter_25_Interview_Prep.md) | Comprehensive Q&A across all frameworks |

---

## Quick Reference — Framework Summary

| Framework | Subsystem | User-Space Interface | Kernel Directory |
|-----------|-----------|---------------------|-----------------|
| V4L2 | Video/Camera | /dev/video* | drivers/media/ |
| ALSA | Audio | /dev/snd/* | sound/ |
| DRM/KMS | Display/GPU | /dev/dri/* | drivers/gpu/drm/ |
| Input | Input devices | /dev/input/event* | drivers/input/ |
| Networking | Network | socket API | net/, drivers/net/ |
| USB | USB devices | /dev/bus/usb/* | drivers/usb/ |
| I2C | I2C bus | /dev/i2c-* | drivers/i2c/ |
| SPI | SPI bus | /dev/spidev* | drivers/spi/ |
| GPIO | GPIO pins | /dev/gpiochip* | drivers/gpio/ |
| DMA | DMA transfers | (kernel-only API) | drivers/dma/ |
| Clock | Clock tree | (kernel-only API) | drivers/clk/ |
| Regulator | Voltage regs | (kernel-only API) | drivers/regulator/ |
| Power Mgmt | PM states | /sys/power/* | drivers/base/power/ |
| Thermal | Temperature | /sys/class/thermal/ | drivers/thermal/ |

---

## Reading Paths

**Camera/Video Engineer**: Ch 1 → 2 → 3 → 9 → 12 → 17 → 18 → 21 → 25
**Audio Engineer**: Ch 1 → 2 → 4 → 9 → 10 → 13 → 17 → 25
**Display/GPU Engineer**: Ch 1 → 2 → 5 → 12 → 13 → 17 → 25
**BSP/Platform Engineer**: Ch 1 → 2 → 9 → 10 → 11 → 13 → 14 → 15 → 16 → 17 → 22 → 25
**Automotive Engineer**: Ch 1 → 2 → 3 → 4 → 5 → 7 → 15 → 17 → 22 → 23 → 25
**Interview Preparation**: Ch 1 → 2 → 18 → 19 → 20 → 25

---

*Each chapter includes: Learning Goals, Numbered Sections with ASCII Diagrams, Real Kernel C Code, Kernel Source References, Interview Questions (Q&A), Summary, and Next Chapter Link.*
