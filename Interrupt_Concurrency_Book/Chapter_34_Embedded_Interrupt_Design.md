# Chapter 34: Embedded System Interrupt Design

## Learning Goals
- Design interrupt systems for resource-constrained embedded platforms
- Handle SoC-specific interrupt controllers (GIC, NVIC, VIC)
- Apply power-aware interrupt strategies
- Manage real-time constraints in embedded Linux
- Deal with multi-core SoC interrupt routing

---

## 34.1 Embedded Interrupt Landscape

```
Embedded Platform Spectrum:

  Bare-metal MCU          RTOS + MPU           Linux SoC (Application Processor)
  ┌─────────────┐         ┌─────────────┐      ┌───────────────────────┐
  │ ARM Cortex-M│         │ ARM Cortex-R│      │ ARM Cortex-A + GIC    │
  │ NVIC (240   │         │ VIC / GIC   │      │ SA8155P, i.MX8, etc.  │
  │  interrupts)│         │ FreeRTOS    │      │ Linux + DT            │
  │ No OS / RTOS│         │ QNX / Zephyr│      │ Threaded IRQ          │
  └─────────────┘         └─────────────┘      └───────────────────────┘
       │                       │                        │
       └──── Simple ────── Medium ────── Complex ───────┘
```

### SoC Interrupt Architecture (Typical Automotive)

```
  ┌────────────────────────────────────────────────────┐
  │                  SoC (e.g., SA8155P)               │
  │                                                    │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
  │  │ Cortex-A │  │ Cortex-A │  │ Cortex-A │ (app)  │
  │  │ CPU 0    │  │ CPU 1    │  │ CPU 2-7  │        │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
  │       │              │              │              │
  │  ┌────┴──────────────┴──────────────┴────┐        │
  │  │        GIC-500 (GICv3)                │        │
  │  │  Distributor → Redistributor per CPU  │        │
  │  │  SPI: 0-1023 (shared peripheral IRQs) │        │
  │  │  PPI: 16-31 (per-CPU private IRQs)    │        │
  │  │  SGI: 0-15 (software generated / IPI) │        │
  │  └──┬───────────────────────────────────┘         │
  │     │                                              │
  │  ┌──┴────────────────────────────────────────┐    │
  │  │            Peripheral IRQs                 │    │
  │  │  UART  SPI  I2C  GPIO  DMA  Timer  USB    │    │
  │  │  CAN   PCIe  Display  Camera  NPU  DSP    │    │
  │  └───────────────────────────────────────────┘    │
  └────────────────────────────────────────────────────┘
```

---

## 34.2 Device Tree Interrupt Specification

On embedded Linux, interrupts are described in Device Tree:

```dts
/* GIC interrupt controller: */
gic: interrupt-controller@17a00000 {
    compatible = "arm,gic-v3";
    #interrupt-cells = <3>;
    interrupt-controller;
    reg = <0 0x17a00000 0 0x10000>,    /* Distributor */
          <0 0x17a60000 0 0x100000>;   /* Redistributor */
};

/* Device using interrupts: */
uart0: serial@a84000 {
    compatible = "qcom,msm-uartdm";
    reg = <0 0xa84000 0 0x200>;
    interrupts = <GIC_SPI 108 IRQ_TYPE_LEVEL_HIGH>;
    /*             type  number  trigger */
    clocks = <&gcc GCC_UART_CLK>;
};

/* GPIO controller as interrupt controller: */
tlmm: pinctrl@3400000 {
    compatible = "qcom,sa8155p-tlmm";
    reg = <0 0x03400000 0 0x400000>;
    interrupts = <GIC_SPI 208 IRQ_TYPE_LEVEL_HIGH>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
};

/* Device using GPIO interrupt: */
sensor@48 {
    compatible = "ti,tmp102";
    reg = <0x48>;
    interrupt-parent = <&tlmm>;
    interrupts = <25 IRQ_TYPE_EDGE_FALLING>;
};
```

### DT Interrupt Cells

```
GIC (#interrupt-cells = <3>):
  Cell 1: Type (0=SPI, 1=PPI)
  Cell 2: IRQ number
  Cell 3: Trigger flags (1=rising, 2=falling, 4=high, 8=low)

GPIO (#interrupt-cells = <2>):
  Cell 1: GPIO pin number
  Cell 2: Trigger flags
```

---

## 34.3 Power-Aware Interrupt Design

### Wakeup Interrupts

```c
/* Mark IRQ as wake source (can wake from suspend): */
enable_irq_wake(irq);

/* In driver: */
static int my_suspend(struct device *dev)
{
    struct my_dev *d = dev_get_drvdata(dev);

    if (device_may_wakeup(dev))
        enable_irq_wake(d->irq);

    /* Disable non-essential interrupts: */
    disable_irq(d->data_irq);

    return 0;
}

static int my_resume(struct device *dev)
{
    struct my_dev *d = dev_get_drvdata(dev);

    if (device_may_wakeup(dev))
        disable_irq_wake(d->irq);

    enable_irq(d->data_irq);
    return 0;
}

/* DT: mark device as wakeup source */
device@addr {
    wakeup-source;
    interrupts = <GIC_SPI 42 IRQ_TYPE_EDGE_RISING>;
};
```

### Power Domain Considerations

```
Problem: Device in power-off domain → IRQ line floating → spurious IRQs

Solution:
  1. Disable IRQ before power-gating the domain
  2. Use wakeup-capable IRQs routed through always-on controller
  3. Some SoCs have dedicated "wake-up interrupt controller" (WIC)

  ┌─────────────┐     ┌──────────────┐
  │ Always-ON   │     │ Power Domain │
  │ controller  │     │ (can be OFF) │
  │ ┌─────────┐ │     │ ┌──────────┐ │
  │ │ WIC     │◄├─────┤ │ Device   │ │
  │ │ (wakeup)│ │     │ │ IRQ      │ │
  │ └────┬────┘ │     │ └──────────┘ │
  │      │      │     └──────────────┘
  │      ▼      │
  │ Wake CPU    │
  │ + restore   │
  │ power domain│
  └─────────────┘
```

---

## 34.4 Interrupt Latency Budget

For real-time embedded systems, every microsecond counts:

```
Interrupt Latency Budget (automotive camera example):

  Frame rate: 30 fps → 33.3ms per frame
  ISR deadline: < 100µs after frame-ready IRQ

  Component                    │ Budget (µs)
  ─────────────────────────────┼────────────
  GIC routing + CPU response   │      1-5
  Context save (assembly)      │      1-2
  IRQ subsystem overhead       │      2-5
  Driver handler (top half)    │     10-30
  DMA setup for frame          │      5-10
  ─────────────────────────────┼────────────
  Total                        │     19-52µs ✓ (<100µs)

  Bottom half (deferred):
  Frame processing              │   1000-5000
  Display update                │    500-2000
```

### Measuring on Embedded

```bash
# GPIO-based measurement (most accurate):
# Toggle GPIO at ISR entry and exit, measure with oscilloscope

static irqreturn_t my_isr(int irq, void *data)
{
    gpio_set_value(DEBUG_GPIO, 1);     /* Scope trigger */
    /* ... handler work ... */
    gpio_set_value(DEBUG_GPIO, 0);     /* Scope end */
    return IRQ_HANDLED;
}

# ftrace on embedded:
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... run workload ...
cat /sys/kernel/debug/tracing/trace
```

---

## 34.5 Multi-Core SoC Interrupt Strategy

```
Typical automotive SoC CPU assignment:

  CPU 0-1: Android UI / infotainment (non-RT)
  CPU 2-3: ADAS / camera processing (soft RT)
  CPU 4-5: Vehicle control (hard RT, isolated)
  CPU 6-7: AI/ML inference (batch)

IRQ assignment:
  Display IRQ    → CPU 0 (UI)
  Audio IRQ      → CPU 1 (media)
  Camera IRQ     → CPU 2 (ADAS)
  CAN bus IRQ    → CPU 4 (vehicle control)
  Ethernet IRQ   → CPU 3 (networking)
  DMA IRQs       → Same CPU as requestor (NUMA-like)
```

```bash
# Set up isolated CPUs at boot:
isolcpus=4,5 nohz_full=4,5 rcu_nocbs=4,5

# Pin IRQs:
echo 4 > /proc/irq/XXX/smp_affinity_list  # CAN to CPU 4

# Pin RT threads:
taskset -c 4 chrt -f 90 ./vehicle_control
```

---

## 34.6 Shared Resources in Multi-Core

```c
/* Inter-processor communication via shared memory + IRQ: */

struct shared_region {
    atomic_t    cpu0_to_cpu4_flag;
    atomic_t    cpu4_to_cpu0_flag;
    u8          data[4096];
} __attribute__((aligned(64)));

/* CPU 0: Send data to CPU 4 */
memcpy(shared->data, buffer, len);
smp_wmb();                              /* Data before flag */
atomic_set(&shared->cpu0_to_cpu4_flag, 1);
/* Trigger IPI or doorbell interrupt to CPU 4 */

/* CPU 4: ISR receives notification */
static irqreturn_t doorbell_isr(int irq, void *data)
{
    if (atomic_read(&shared->cpu0_to_cpu4_flag)) {
        smp_rmb();                      /* Flag before data */
        process(shared->data);
        atomic_set(&shared->cpu0_to_cpu4_flag, 0);
    }
    return IRQ_HANDLED;
}
```

---

## 34.7 Common Embedded Peripherals and IRQ Patterns

```
Peripheral  │ IRQ Pattern           │ Notes
────────────┼───────────────────────┼──────────────────────
UART        │ RX FIFO threshold     │ Threaded IRQ for I/O
            │ TX empty              │
SPI/I2C     │ Transfer complete     │ Threaded (bus is slow)
GPIO        │ Edge/level change     │ Debounce in software
DMA         │ Transfer complete     │ Top-half: complete()
Timer       │ Period elapsed        │ hrtimer or hw timer
CAN         │ Message received      │ NAPI-like buffering
Camera/ISP  │ Frame done, vsync     │ DMA + completion
Display     │ Vsync, underrun       │ Top-half: schedule work
Watchdog    │ Timer expire (NMI)    │ Reset or recovery
Thermal     │ Threshold crossed     │ Threaded: throttle
USB         │ Endpoint events       │ Complex state machine
PCIe        │ MSI/MSI-X events      │ Per-function vector
```

---

## 34.8 Low-Power Interrupt Optimization

```c
/* Reduce interrupt rate for battery-powered devices: */

/* 1. Use interrupt coalescing: */
set_coalesce_timer(dev, 1000);  /* Batch events for 1ms */

/* 2. Use level-triggered for wake (more reliable in low-power): */
interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;

/* 3. Use threaded IRQ (CPU can idle between handler calls): */
request_threaded_irq(irq, NULL, handler_fn, IRQF_ONESHOT, ...);

/* 4. Disable unused IRQs: */
disable_irq(unused_irq);

/* 5. Use wakeup-capable IRQ lines: */
device_init_wakeup(dev, true);
enable_irq_wake(irq);

/* 6. Adjust polling vs interrupt based on load:
   Low traffic: interrupt mode (CPU sleeps between events)
   High traffic: polling mode (avoid IRQ overhead) */
```

---

## 34.9 Debugging Embedded Interrupts

```
Tool                          │ Use Case
──────────────────────────────┼───────────────────────────────
GPIO toggle + oscilloscope    │ Precise timing measurement
JTAG debugger                 │ Breakpoint in ISR, register dump
/proc/interrupts via adb      │ Count verification on Android
ftrace via tracefs             │ Latency tracing
Serial console (earlycon)     │ Boot-time IRQ issues
devmem2                       │ Read HW interrupt registers
GIC register dump             │ Routing/priority verification
Logic analyzer                │ Multi-signal correlation
```

```bash
# Android embedded debug:
adb shell cat /proc/interrupts
adb shell "echo irqsoff > /sys/kernel/debug/tracing/current_tracer"

# Read GIC distributor register (check enable):
devmem2 0x17A00100 w   # GICD_ISENABLER: which IRQs enabled

# Check interrupt pending:
devmem2 0x17A00200 w   # GICD_ISPENDR: which IRQs pending
```

---

## Interview Questions

1. **How do you describe interrupts in Device Tree?**
2. **What is a wakeup interrupt? How do you configure one?**
3. **How do you assign IRQs to specific CPUs on a multi-core SoC?**
4. **What is the interrupt latency budget for a real-time embedded system?**
5. **How do you debug a missing interrupt on an ARM SoC?**
6. **Explain GIC SPI vs PPI vs SGI.**
7. **How do you handle interrupts across power domains?**
8. **Design the interrupt architecture for an automotive camera subsystem.**
9. **What tools do you use to measure interrupt latency on embedded?**
10. **How does isolcpus help embedded real-time systems?**

---

## Summary

- Embedded SoCs use GIC (ARM) with Device Tree to describe IRQ topology
- Power management: wake IRQs, power domain awareness, coalescing
- Multi-core strategy: isolate CPUs, pin IRQs, assign priorities
- Latency budget: account for every component from hardware to handler
- Use threaded IRQs for I2C/SPI peripherals (slow bus, need sleep)
- Debug with GPIO toggle + scope for precise timing, ftrace for software
- Device Tree specifies interrupt type, number, and trigger for each device
- Low-power: balance interrupt rate vs CPU idle time

---

*Next: [Chapter 35 — Documentation and References](Chapter_35_Documentation_References.md)*
