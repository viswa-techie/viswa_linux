# Chapter 25: Interview Preparation — Linux Device Frameworks

## Learning Goals
- Master the 50 most-asked interview questions on Linux device frameworks
- Know how to explain framework concepts clearly under pressure
- Practice tracing end-to-end flows verbally
- Prepare for system design questions involving framework selection

---

## 25.1 Foundational Questions

**Q1: What is a Linux device framework? Give examples.**
A: A framework provides a standardized API between user space, kernel core, and hardware drivers for a specific device class. It handles common functionality so individual drivers focus only on hardware-specific code. Examples: V4L2 (video), ALSA/ASoC (audio), DRM/KMS (display), input subsystem, netdev (network), I2C, SPI, GPIO. Each framework defines: (1) user-space API (ioctl/sysfs/char device), (2) core management code (device registration, matching, dispatch), (3) driver callbacks (ops structures the driver must implement).

**Q2: Explain the device model: bus, device, driver.**
A: `struct bus_type` represents a bus (I2C, SPI, platform). `struct device` represents a hardware instance. `struct device_driver` represents code that can drive a device. When a device appears on a bus, the bus's `match()` function checks if any registered driver can handle it (comparing compatible strings, IDs). If matched, the driver's `probe()` is called with the device. This decouples device discovery from driver code — the same driver handles any matching device regardless of how it was discovered (DT, ACPI, hotplug).

**Q3: What is the purpose of device tree in framework drivers?**
A: Device tree describes hardware in a structured, portable format. It specifies: device addresses (`reg`), interrupts, clock references (`clocks`), power supplies (`*-supply`), GPIOs (`*-gpios`), DMA channels (`dmas`), and connections between devices (phandles). Framework drivers parse these properties using standardized APIs (`devm_clk_get()`, `devm_regulator_get()`). DT eliminates board-specific C code and makes the same driver work across different boards by changing only the DT file.

---

## 25.2 Bus Framework Questions

**Q4: Compare I2C and SPI. When do you use each?**
A: I2C: 2-wire (SDA + SCL), half-duplex, multi-slave with addresses, up to 3.4 MHz. Use for: low-speed devices sharing a bus (sensors, PMICs, EEPROMs, RTCs). SPI: 4+ wire (MOSI, MISO, SCLK, CS), full-duplex, CS selects slave, up to 100+ MHz. Use for: high-speed data (NOR flash, ADCs, displays). Key differences: SPI needs one CS pin per slave (uses PCB resources); I2C uses addresses (2-wire for many devices). SPI has no ACK mechanism; I2C has per-byte ACK.

**Q5: How does I2C device matching work?**
A: Three mechanisms: (1) **Device Tree**: I2C adapter's DT node has child nodes with `compatible` and `reg` (address). The I2C core creates `i2c_client` for each child, matches against driver's `of_match_table`. (2) **id_table**: Driver registers `i2c_device_id` with device names matched against board-registered info. (3) **ACPI**: Similar to DT with ACPI device IDs. On match, `probe()` is called with `i2c_client` containing the adapter and address.

**Q6: What is regmap and why should drivers use it?**
A: regmap abstracts register access over any bus (I2C, SPI, MMIO) with a uniform API: `regmap_read()`, `regmap_write()`, `regmap_update_bits()`. Benefits: (1) Bus-agnostic — same driver code for I2C and SPI variants. (2) Register caching — avoids redundant I2C reads. (3) Atomic read-modify-write with locking. (4) Debugfs integration for register dump. (5) Range checking. A driver using regmap can be ported to a different bus by changing only the initialization (`devm_regmap_init_i2c()` → `devm_regmap_init_spi()`).

---

## 25.3 Media/Display Framework Questions

**Q7: Explain V4L2 buffer management (videobuf2).**
A: vb2 manages buffers for video capture/output. Flow: (1) `VIDIOC_REQBUFS(count=4)` — vb2 calls driver's `queue_setup()` to allocate DMA-capable buffers. (2) `VIDIOC_QBUF(index)` — queues buffer to driver, `buf_queue()` submits DMA address to HW. (3) `VIDIOC_STREAMON` — `start_streaming()` begins DMA. (4) HW fills buffer → ISR calls `vb2_buffer_done()`. (5) `VIDIOC_DQBUF` — dequeues filled buffer to user space. Memory types: MMAP (kernel allocates, user mmaps), USERPTR (user allocates), DMABUF (fd-based sharing).

**Q8: What is the media controller and why is it needed?**
A: Simple cameras use one video device (/dev/video0). Complex pipelines (sensor → CSI → ISP → capture) need the media controller. Each component is a `media_entity` with pads (ports). `media_link` connects pads. User space uses /dev/media0 to discover topology and configure links at runtime. Without it, there's no way to express which sensor feeds which ISP configuration, or switch between front/rear cameras in a SoC with multiple CSI ports.

**Q9: Explain DRM atomic modesetting.**
A: Atomic modesetting commits all display changes (plane, CRTC, connector) in a single ioctl. User space builds a request with property changes, submits it. Kernel: (1) `atomic_check()` validates all changes — bandwidth, formats, plane overlap. (2) If `TEST_ONLY` flag, return result without applying. (3) `atomic_commit()` applies all changes at next VBlank — no tearing. Benefits over legacy: no partial state, validation before commit, rollback on failure, test-only mode.

**Q10: How would you debug a blank display?**
A: (1) Check `/dev/dri/card*` exists. (2) `cat /sys/kernel/debug/dri/0/state` — shows CRTC/plane/connector state. (3) `modetest -c` — check connector status (connected?). (4) `modetest -s <conn>@<crtc>:WxH` — test pattern. (5) Check clocks: `clk_summary | grep pixel`. (6) Check regulators: `regulator_summary`. (7) Enable `drm.debug=0x1f`. (8) Check encoder/PHY initialization (DSI/LVDS). (9) Check panel reset GPIO and enable timing.

---

## 25.4 Audio Framework Questions

**Q11: Explain the ASoC three-driver model.**
A: ASoC separates embedded audio into three reusable drivers: (1) **Machine driver** — board-specific, defines which codec connects to which CPU DAI (I2S port) via `snd_soc_dai_link`. Handles board-level wiring: clock routing, amplifier GPIOs, jack detection. (2) **Platform/CPU DAI driver** — SoC-specific, manages I2S/TDM controller hardware and DMA engine. (3) **Codec driver** — codec chip-specific, controls DAC/ADC via I2C/SPI registers, mixer controls, DAPM widgets. The machine driver glues the other two, enabling reuse: same codec driver across SoCs, same platform driver with different codecs.

**Q12: What is DAPM?**
A: Dynamic Audio Power Management. DAPM automatically powers audio path widgets (DAC, mixer, amplifier) based on active audio routes. When playback starts, DAPM traces the active path from I2S input to headphone output and powers only those widgets. When playback stops, it powers them down. Rules are defined by routes: `{sink, control, source}`. This is transparent to user space — no manual power management calls needed. Critical for battery devices where a codec might have 20+ power domains but only 5 are needed for headphone playback.

---

## 25.5 Power and Resource Framework Questions

**Q13: Compare system suspend and Runtime PM.**
A: System suspend: all devices suspended, triggered by user/policy (`echo mem > /sys/power/state`). System enters low-power state (S3). Runtime PM: per-device, automatic. Each device independently suspends when idle (`pm_runtime_put_autosuspend()`) and resumes on demand (`pm_runtime_get_sync()`). System stays running. Runtime PM is more granular — camera can be suspended while display is active. Both use similar callbacks (save state, gate clocks), but Runtime PM operates continuously during normal operation.

**Q14: What is autosuspend in Runtime PM?**
A: Autosuspend adds a configurable delay between the last `pm_runtime_put()` and actually invoking `runtime_suspend()`. Without it, a device handling brief requests (I2C reads) would suspend/resume for every request — the power overhead of transitions would exceed the savings. With autosuspend (e.g., 200ms), the device stays powered if another request arrives within the delay window. `pm_runtime_mark_last_busy()` resets the timer. This amortizes transition costs over bursty workloads.

**Q15: How does the clock framework handle prepare vs enable?**
A: Two-phase design: `clk_prepare()` may sleep — powers up PLL, waits for lock, enables power domain. Called from process context only. `clk_enable()` is atomic — opens the clock gate register. Can be called from interrupt context. This allows drivers in ISR to call `clk_enable()` if `clk_prepare()` was done earlier. Both are reference-counted: the clock is only actually disabled when ALL users have called disable/unprepare. `clk_prepare_enable()` combines both for convenience in non-atomic context.

**Q16: How does the regulator framework handle multiple consumers?**
A: Reference-counted enable: the regulator stays on until ALL consumers have called `regulator_disable()`. Voltage arbitration: if consumer A requests 2.8-3.0V and consumer B requests 2.5-3.3V, the framework selects voltage in the intersection (2.8-3.0V). Device tree constraints (`regulator-min/max-microvolt`) define the hardware-safe range. `regulator-always-on` prevents any consumer from disabling the supply. This ensures no consumer's voltage requirements are violated.

---

## 25.6 Input and GPIO Questions

**Q17: Explain Linux input event model.**
A: Input devices report events as (type, code, value) tuples. Types: EV_KEY (buttons/keys), EV_ABS (touchscreen coordinates), EV_REL (mouse movement), EV_SW (switches). The driver calls `input_report_key()`, `input_report_abs()`, etc., then `input_sync()` which sends `SYN_REPORT`. User space reads struct `input_event` from `/dev/input/eventN`. SYN_REPORT marks a complete packet — all events between two SYN_REPORTs belong to the same hardware sample.

**Q18: Compare multitouch Type A and Type B.**
A: Type A: anonymous contacts — each sends coordinates + SYN_MT_REPORT, no tracking. User space must track fingers. Type B: slot-based — `input_mt_slot(id)` selects a numbered slot, per-slot data reported. Kernel tracks fingers by slot ID across frames. Type B supports partial updates (only changed slots). Type B is required by Android and is the standard for modern touchscreens.

**Q19: What does can_sleep mean in GPIO controllers?**
A: `can_sleep = false`: GPIO get/set can be called from atomic context (IRQ handlers, spinlocks). For memory-mapped (MMIO) GPIO controllers where register access is a simple writel/readl. `can_sleep = true`: get/set may sleep. For GPIO controllers on I2C/SPI buses (e.g., PCA9535 I2C GPIO expander), because bus transactions require sleeping. Consumers must use `gpiod_get_value_cansleep()` for sleeping controllers.

---

## 25.7 DMA and Network Questions

**Q20: Explain coherent vs streaming DMA mappings.**
A: Coherent (`dma_alloc_coherent()`): allocates memory that's always consistent between CPU and device. Typically uncached or write-combined. No sync needed but slower CPU access. Used for descriptor rings and small control structures. Streaming (`dma_map_single/sg()`): maps existing memory for DMA. Memory can be cached for performance. Requires explicit sync: `dma_sync_*_for_device()` before device reads, `dma_sync_*_for_cpu()` after device writes. Used for large data buffers (network packets, video frames).

**Q21: What is NAPI and how does it work?**
A: NAPI (New API) converts network RX from interrupt-driven to polling under load. Flow: (1) First packet arrives → hardware interrupt. (2) ISR disables further interrupts, calls `napi_schedule()`. (3) Softirq runs the poll function, processing up to `budget` packets without interrupts. (4) If all packets processed, re-enable interrupts. (5) If budget hit, stay in polling mode. This prevents interrupt storms at high packet rates (millions/sec) where interrupt overhead would consume 100% CPU.

**Q22: Explain the sk_buff structure.**
A: sk_buff is the network buffer for one packet. Four pointers: `head` (buffer start), `data` (current data start), `tail` (data end), `end` (buffer end). `skb_push(len)`: add header (data moves backward). `skb_pull(len)`: strip header (data moves forward). `skb_put(len)`: extend data (tail moves forward). `skb_reserve(len)`: create headroom. The buffer layout after receive: headroom → Ethernet header → IP header → TCP header → payload → tailroom.

---

## 25.8 USB Questions

**Q23: Explain USB enumeration.**
A: (1) Device connects — hub detects. (2) Hub resets port. (3) Host sends GET_DESCRIPTOR to address 0 — reads device descriptor (VID/PID/class). (4) SET_ADDRESS — assigns unique bus address. (5) Host reads full descriptors (configuration, interfaces, endpoints). (6) SET_CONFIGURATION selects the active configuration. (7) USB core matches against registered drivers using VID/PID or class/subclass/protocol. (8) Matching driver's `probe()` is called.

**Q24: What are URBs?**
A: URB (USB Request Block) is the fundamental USB I/O unit. Lifecycle: (1) `usb_alloc_urb()` — allocate. (2) `usb_fill_int_urb()` / `usb_fill_bulk_urb()` — configure endpoint, buffer, size, callback. (3) `usb_submit_urb()` — submit to HCD for scheduling. (4) HCD executes transfer on bus. (5) Completion callback fires with status. (6) For continuous polling, callback re-submits URB. (7) `usb_kill_urb()` cancels on disconnect.

---

## 25.9 System Design Questions

**Q25: Design the software stack for a rearview camera system.**
A: Hardware: Camera sensor → MIPI CSI-2 → SoC ISP → display. Software stack: (1) **I2C**: configure camera sensor registers. (2) **Clock**: provide MCLK to sensor. (3) **Regulator**: power sensor AVDD/DVDD. (4) **GPIO**: sensor reset/enable. (5) **V4L2**: capture driver with media controller for pipeline. (6) **DMA**: frame transfer from ISP to memory. (7) **DMA-buf**: export frame as fd. (8) **DRM**: import fd, display on overlay plane. (9) **Runtime PM**: power down when not in reverse gear. (10) Trigger: CAN bus gear position → V4L2 streamon → DRM plane enable.

**Q26: How would you design multi-zone audio for an automotive system?**
A: (1) ASoC card with 4+ DAI links: driver zone (I2S0 → codec0), passenger zone (I2S1 → codec1), rear zone (TDM0 → amp), hands-free (I2S2 → mic array). (2) Each zone has independent volume and routing. (3) Audio policy: phone call ducks media in driver zone, keeps rear zone unchanged. Navigation mixes with media. Emergency alert overrides all zones. (4) DAPM manages per-zone power — rear zone can be powered down when no passengers. (5) Clocks: each I2S has independent clock. (6) Regulators: per-codec power supplies.

**Q27: How do you debug a camera system that produces corrupted frames?**
A: Systematic approach: (1) Check I2C communication — `i2cget` to verify sensor is responding. (2) Verify sensor configuration — read back format/resolution registers. (3) Check DMA: enable `CONFIG_DMA_API_DEBUG`, check `/sys/kernel/debug/dma-api/errors`. (4) Verify buffer alignment and cache coherency — ensure `dma_sync_*` calls present if using streaming DMA. (5) Check frame size matches allocated buffer size (width × height × bytes_per_pixel). (6) Use `v4l2-ctl --stream-mmap` to capture raw frames and inspect. (7) Compare captured resolution format with expected. (8) Check MIPI CSI-2 lane configuration and clock rates.

---

## 25.10 Advanced Questions

**Q28: What is deferred probing?**
A: When a driver requests a resource whose provider hasn't registered yet, the framework returns `-EPROBE_DEFER`. The kernel defers the device and retries later. After all initial probes, the kernel processes the deferred queue repeatedly. This automatically resolves dependency ordering without explicit init ordering. Example: a sensor driver probing before the PMIC driver — `regulator_get()` returns -EPROBE_DEFER, then succeeds after PMIC probes.

**Q29: How does DMA-buf enable zero-copy?**
A: The exporter (e.g., V4L2 camera) allocates DMA-capable memory and creates a `dma_buf` with `dma_buf_export()`. It generates a file descriptor with `dma_buf_fd()`. User space passes this fd to the importer (e.g., DRM display). The importer calls `dma_buf_get(fd)` → `dma_buf_attach()` → `dma_buf_map_attachment()` to get the `sg_table` (scatter-gather table) for its DMA engine. Both devices access the same physical memory — no CPU copy occurs. Fences synchronize producer/consumer access.

**Q30: Explain the thermal control loop.**
A: (1) Thermal sensor reads temperature (thermal zone `get_temp()`). (2) Thermal core compares against trip points. (3) If exceeded, the governor decides cooling action. step_wise: incrementally increase cooling state. power_allocator: PID-based power budget. (4) Cooling device adjusts: cpufreq reduces CPU frequency, fan speeds up. (5) Loop repeats at `polling-delay` interval (faster during active cooling via `polling-delay-passive`). (6) Critical trip: immediate `orderly_poweroff()`. Hysteresis prevents oscillation.

**Q31: What is container_of and where is it used?**
A: `container_of(ptr, type, member)` returns a pointer to the enclosing structure given a pointer to a member. Calculated as: `(type *)((char *)ptr - offsetof(type, member))`. Used throughout the kernel because frameworks define callbacks with generic types (`struct device *`), but drivers need their specific data. By embedding `struct device` inside `struct my_driver`, the driver recovers its structure in callbacks. Every `to_*` macro (to_platform_device, to_i2c_client) uses container_of internally.

---

## 25.11 Quick Reference Table

```
Framework Quick Reference:

┌─────────────┬─────────────────────────┬──────────────────────────┐
│ Framework   │ Key Structure           │ User Space               │
├─────────────┼─────────────────────────┼──────────────────────────┤
│ V4L2        │ video_device, vb2_queue │ /dev/video0, v4l2-ctl    │
│ DRM/KMS     │ drm_device, drm_crtc   │ /dev/dri/card0, modetest │
│ ALSA/ASoC   │ snd_soc_card           │ /dev/snd/*, aplay        │
│ Input       │ input_dev              │ /dev/input/event0, evtest│
│ Network     │ net_device, sk_buff    │ eth0, ip, ethtool        │
│ USB         │ usb_interface, urb     │ /dev/bus/usb, lsusb      │
│ I2C         │ i2c_client, i2c_adapter│ /dev/i2c-N, i2cdetect    │
│ SPI         │ spi_device             │ /dev/spidevN.N           │
│ GPIO        │ gpio_desc, gpio_chip   │ /dev/gpiochipN, gpioget  │
│ DMA         │ dma_chan                │ /sys/class/dma/          │
│ Clock       │ clk, clk_hw            │ debugfs/clk/clk_summary  │
│ Regulator   │ regulator, regulator_dev│ debugfs/regulator/       │
│ PM          │ dev_pm_ops             │ /sys/power/state         │
│ Thermal     │ thermal_zone_device    │ /sys/class/thermal/      │
└─────────────┴─────────────────────────┴──────────────────────────┘
```

---

## 25.12 System Design Interview Template

```
When asked "Design a Linux driver for X":

1. Identify frameworks needed:
   "This requires [V4L2/DRM/ALSA/...] framework because..."

2. Device Tree:
   "The DT node would have: compatible, reg, clocks,
    *-supply, *-gpios, interrupts, dmas"

3. Probe function:
   "In probe: devm_clk_get, devm_regulator_get,
    devm_gpiod_get, devm_platform_ioremap_resource,
    power-up sequence, register with framework"

4. Data path:
   "Data flows: [sensor → CSI → ISP → DMA → memory]
    or [memory → DMA → I2S → codec → speaker]"

5. Power management:
   "Runtime PM with autosuspend: gate clocks when idle,
    system suspend saves HW state"

6. Error handling:
   "All resources via devm_, error paths return cleanly,
    EPROBE_DEFER handled"

7. Debug approach:
   "debugfs for state, ftrace for flow, devmem for registers,
    framework tools (v4l2-ctl, modetest, amixer)"
```

---

## Summary

- Master the three pillars: framework architecture, driver implementation, debugging
- Know end-to-end data flows for camera, audio, display, network
- Understand cross-framework interactions (DMA-buf, deferred probe, power sequencing)
- Be prepared for system design: identify frameworks → DT → probe → data path → PM → debug
- container_of, devm_, regmap, Runtime PM are must-know patterns
- Practice explaining concepts clearly — interviewers test understanding depth, not memorization

---

*This concludes the Linux Device Framework Book.*
*Return to: [Master Index](00_Master_Index.md)*
