# Chapter 22: Embedded and Automotive Architecture

## Learning Goals
- Understand how Linux device frameworks apply in automotive (AAOS, AGL)
- Know automotive-specific framework patterns and constraints
- Grasp safety-critical, real-time, and multi-display architectures
- Design framework interactions for automotive ECUs

---

## 22.1 Automotive Linux Architecture

```
Automotive Linux Stack (AAOS / AGL):

┌─────────────────────────────────────────────────────┐
│  Applications                                        │
│  ├── Instrument Cluster (gauges, speed, warnings)    │
│  ├── IVI (infotainment, nav, media)                  │
│  ├── ADAS (camera feed, parking assist)              │
│  ├── HVAC control                                    │
│  └── OTA update manager                              │
├─────────────────────────────────────────────────────┤
│  Android Framework / AGL Framework                   │
│  ├── SurfaceFlinger / Wayland (multi-display)        │
│  ├── AudioFlinger / PipeWire                         │
│  ├── VHAL (Vehicle HAL — CAN interface)              │
│  ├── Camera Service                                  │
│  └── Power Manager                                   │
├─────────────────────────────────────────────────────┤
│  HAL / Native Libraries                              │
│  ├── HIDL/AIDL services                              │
│  ├── OpenGL ES / Vulkan (GPU)                        │
│  └── Codec2 (media decode)                           │
├─────────────────────────────────────────────────────┤
│  Linux Kernel — Device Frameworks                    │
│  ├── DRM/KMS: multi-display (cluster + IVI + HUD)   │
│  ├── V4L2: rearview, surround-view cameras           │
│  ├── ALSA/ASoC: audio zones (driver, passenger)      │
│  ├── CAN (SocketCAN): vehicle bus communication      │
│  ├── GPIO: button inputs, indicator outputs          │
│  ├── I2C/SPI: sensors, PMIC, touch controllers      │
│  ├── USB: Android Auto, CarPlay, storage             │
│  ├── Thermal: SoC throttling in cabin temps          │
│  └── Power: ignition states, deep sleep              │
├─────────────────────────────────────────────────────┤
│  Hardware: Automotive SoC                            │
│  ├── Qualcomm SA8155P / SA8295P                      │
│  ├── Renesas R-Car H3/M3                             │
│  ├── NXP i.MX 8QuadMax                               │
│  └── TI Jacinto (TDA4)                               │
└─────────────────────────────────────────────────────┘
```

---

## 22.2 Multi-Display Architecture

```
Automotive Multi-Display (DRM/KMS):

SA8155P Example — 4 display outputs:

┌────────────────────────────────────────────────┐
│  DRM Device (/dev/dri/card0)                    │
│                                                 │
│  CRTC 0 ──► DSI Encoder ──► DSI Connector ──►  │
│  ├── Primary plane (cluster UI)                 │──► Instrument
│  └── Overlay plane (warning icons)              │    Cluster
│                                                 │
│  CRTC 1 ──► DSI Encoder ──► DSI Connector ──►  │
│  ├── Primary plane (IVI main)                   │──► Center IVI
│  ├── Overlay plane (navigation)                 │    Display
│  └── Overlay plane (rearview camera)            │
│                                                 │
│  CRTC 2 ──► HDMI Encoder ──► HDMI Connector─►  │
│  └── Primary plane (rear seat entertainment)    │──► Rear Display
│                                                 │
│  CRTC 3 ──► LVDS Encoder ──► LVDS Connector─►  │
│  └── Primary plane (HUD projection)            │──► Head-Up
│                                                 │    Display
└────────────────────────────────────────────────┘

Android SurfaceFlinger manages displays:
  Display 0 (PRIMARY)  → Instrument Cluster
  Display 1 (SECONDARY)→ IVI
  Display 2 (EXTERNAL) → Rear Seat
  Display 3 (EXTERNAL) → HUD

Each display independently:
  - Has its own CRTC and timing
  - Runs different content
  - Can have different resolution/refresh rate
  - Uses separate planes for compositing
```

---

## 22.3 Automotive Camera System

```
Surround-View Camera System:

┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│Front │  │ Left │  │Right │  │ Rear │
│Camera│  │Camera│  │Camera│  │Camera│
└──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
   │ MIPI    │ MIPI    │ MIPI    │ MIPI
┌──▼────────▼────────▼────────▼──────┐
│         MIPI CSI-2 Deserializer     │
│    (MAX9296, MAX96712, TI DS90UB)   │
│         Connected via I2C           │
└─────────────────┬───────────────────┘
                  │ MIPI CSI-2 (virtual channels)
┌─────────────────▼───────────────────┐
│            SoC CSI Receiver          │
│     V4L2 subdev + media entities     │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│              ISP                     │
│     V4L2 subdev (processing)         │
│     Outputs 4 streams               │
└─┬───────┬───────┬───────┬──────────┘
  │       │       │       │
  ▼       ▼       ▼       ▼
  4 × /dev/videoN (V4L2 capture)

Frameworks involved:
  V4L2    — camera capture and pipeline
  I2C     — deserializer/serializer configuration
  GPIO    — camera power, reset
  Clock   — pixel clock, reference clock
  Regulator— camera power rails
  DMA     — frame DMA from ISP to memory
  DRM     — overlay plane for rearview display
  DMA-buf — zero-copy camera → display
```

---

## 22.4 Automotive Audio Architecture

```
Automotive Audio Zones:

┌──────────────────────────────────────────────────┐
│  Audio SoC (ASoC)                                 │
│                                                   │
│  Sound Card: "vehicle-audio"                      │
│                                                   │
│  DAI Link 0: CPU I2S0 ──► Codec0 (Driver zone)   │
│  ├── Front left speaker                           │
│  ├── Front right speaker                          │
│  └── Center speaker                               │
│                                                   │
│  DAI Link 1: CPU I2S1 ──► Codec1 (Passenger zone)│
│  ├── Headphone jack (front passenger)             │
│  └── Side speakers                                │
│                                                   │
│  DAI Link 2: CPU TDM0 ──► Amplifier (Rear zone)  │
│  ├── Rear left/right speakers                     │
│  └── Subwoofer                                    │
│                                                   │
│  DAI Link 3: CPU I2S2 ──► Hands-free mic          │
│  └── Microphone array (voice recognition)         │
│                                                   │
│  DAI Link 4: CPU I2S3 ──► USB Audio (Android Auto)│
│                                                   │
│  Mixer controls:                                  │
│  ├── "Navigation Volume"    (priority audio)      │
│  ├── "Media Volume"         (music)               │
│  ├── "Phone Volume"         (Bluetooth call)      │
│  └── "Alert Volume"         (chimes, warnings)    │
│                                                   │
│  Audio routing changes:                           │
│  ├── Phone call → duck media, route to driver     │
│  ├── Navigation → mix with media                  │
│  └── Emergency → mute all, play alert             │
└──────────────────────────────────────────────────┘
```

---

## 22.5 Vehicle Bus Integration (CAN)

```
Vehicle Bus — CAN/LIN/Ethernet:

┌──────────────────────────────────────────────────┐
│  VHAL (Vehicle HAL) — Android/AGL                 │
│  ├── Vehicle speed → cluster display              │
│  ├── Gear position → camera trigger               │
│  ├── Temperature → HVAC control                   │
│  └── Door status → lock/unlock UI                 │
└─────────────────┬────────────────────────────────┘
                  │ HAL interface
┌─────────────────▼────────────────────────────────┐
│  SocketCAN (kernel)                               │
│  ├── socket(AF_CAN, SOCK_RAW, CAN_RAW)           │
│  ├── bind(sock, "can0")                           │
│  ├── read() → struct can_frame                    │
│  ├── write() → send CAN frame                    │
│  └── CAN filters for specific message IDs         │
└─────────────────┬────────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────────┐
│  CAN Controller Driver                            │
│  ├── struct can_priv (bittiming, state)           │
│  ├── net_device_ops (ndo_start_xmit)              │
│  └── Connected via SPI (MCP2515) or MMIO          │
└─────────────────┬────────────────────────────────┘
                  │ CAN bus
┌─────────────────▼────────────────────────────────┐
│  Vehicle CAN Bus                                  │
│  ├── Engine ECU                                   │
│  ├── Transmission ECU                             │
│  ├── Body Control Module                          │
│  ├── ADAS ECU                                     │
│  └── Infotainment (this ECU)                      │
└──────────────────────────────────────────────────┘
```

---

## 22.6 Automotive Power States

```
Automotive Power States:

┌──────────────────────────────────────────────────────┐
│  Ignition States (mapped to Linux PM):               │
│                                                      │
│  OFF (Ignition OFF, doors locked)                    │
│  ├── Deep sleep: DRAM self-refresh, SoC power off    │
│  ├── Only: RTC, CAN wakeup controller, key fob IRQ   │
│  └── Linux: suspend-to-RAM (S3)                      │
│                                                      │
│  ACC (Accessory — radio only)                        │
│  ├── Partial boot: audio + radio active              │
│  ├── Display off, cameras off                        │
│  ├── Runtime PM: most devices suspended              │
│  └── Linux: running, most devices runtime-suspended  │
│                                                      │
│  ON (Ignition ON, engine running)                    │
│  ├── Full system: all displays, cameras, ADAS        │
│  ├── All devices active                              │
│  └── Linux: fully running, all devices active        │
│                                                      │
│  Transitions:                                        │
│  OFF → ACC: CAN wakeup or key fob → resume from S3  │
│  ACC → ON:  Start button → enable remaining devices  │
│  ON → ACC:  Engine off → Runtime PM suspends devices │
│  ACC → OFF: Timer timeout → system suspend (S3)      │
│                                                      │
│  Framework interactions during ON→OFF:               │
│  1. DRM: disable displays, power down panel          │
│  2. V4L2: stop cameras, disable CSI                  │
│  3. ALSA: stop audio, disable codecs                 │
│  4. GPIO: set safe states for outputs                │
│  5. Clock: gate all peripheral clocks                │
│  6. Regulator: disable non-essential supplies        │
│  7. CAN: enter bus-off or listen-only mode           │
│  8. Thermal: stop monitoring                         │
│  9. PM: enter S3 (suspend-to-RAM)                    │
└──────────────────────────────────────────────────────┘
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| SocketCAN | net/can/ | CAN protocol stack |
| CAN drivers | drivers/net/can/ | CAN controller drivers |
| DRM multi-display | drivers/gpu/drm/ | Multi-CRTC support |
| ASoC multi-link | sound/soc/soc-core.c | Multiple DAI links |
| Automotive SoC | drivers/soc/qcom/ | Qualcomm SoC drivers |

---

## Interview Questions

**Q1: How does an automotive system handle multi-display with DRM?**
A: The DRM driver exposes multiple CRTCs, each driving a separate display (cluster, IVI, rear, HUD). Each CRTC connects to its own encoder (DSI, HDMI, LVDS) and connector. Android SurfaceFlinger or Wayland compositor assigns content to each display independently. Atomic modesetting allows configuring all displays in a single commit. Each display can have different resolution, refresh rate, and content. Overlay planes enable hardware compositing — e.g., rearview camera on the IVI display is rendered by a V4L2-to-DRM-plane path using DMA-buf zero-copy.

**Q2: Describe the automotive power state transitions and framework involvement.**
A: Automotive has three main states mapped to PM: OFF (S3 suspend), ACC (partial), ON (full). ON→OFF: DRM disables displays, V4L2 stops cameras, ALSA silences audio, clocks are gated, regulators disabled, CAN enters listen-only mode, then the system enters S3 (suspend-to-RAM). OFF→ACC: CAN wakeup event or key fob triggers resume from S3, but only audio/radio subsystem is fully enabled; cameras and displays stay runtime-suspended. ACC→ON: Remaining devices resume — displays power on, cameras start, ADAS initializes. Each transition requires coordinated framework callbacks in the correct order.

**Q3: How does the surround-view camera system use Linux frameworks?**
A: Four cameras connect via MIPI CSI-2 through a deserializer (MAX96712) which uses virtual channels to multiplex streams. (1) **I2C** configures the deserializer/serializer chain. (2) **V4L2 media controller** represents the pipeline: sensor → serializer → cable → deserializer → CSI → ISP. (3) **V4L2 capture** provides 4 video nodes for the 4 camera streams. (4) **DMA** transfers frames from ISP to memory. (5) **DMA-buf** shares buffers with GPU for surround-view stitching. (6) **DRM overlay plane** displays the stitched view. (7) **GPIO/regulator/clock** manage camera power sequences.

---

## Summary

- Automotive Linux uses all major frameworks: DRM, V4L2, ALSA, CAN, I2C, GPIO, thermal
- Multi-display via DRM: separate CRTCs for cluster, IVI, rear, HUD
- Surround-view cameras: deserializer → CSI → ISP → V4L2 → DMA-buf → DRM
- Audio zones: multiple ASoC DAI links for driver, passenger, rear zones
- CAN bus: SocketCAN integrates vehicle data with Linux networking
- Power states: ignition-mapped transitions with coordinated framework suspend/resume

---

*Next: [Chapter 23 — Embedded Platform Patterns](Chapter_23_Embedded_Platforms.md)*
