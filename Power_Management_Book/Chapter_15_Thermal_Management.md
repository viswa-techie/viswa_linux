# Chapter 15: Thermal Management

## Learning Goals
- Understand Linux thermal framework architecture
- Learn thermal zones, trip points, and cooling devices
- Know thermal governors (step_wise, power_allocator, bang_bang)
- Understand Intelligent Power Allocation (IPA)
- Learn thermal device tree bindings and debugging

---

## 1. Thermal Framework Architecture

```
  Linux Thermal Framework
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  ┌── Thermal Zones ─────────┐   ┌── Cooling Devices ──────┐ │
  │  │  CPU thermal zone         │   │  cpufreq-cooling         │ │
  │  │  GPU thermal zone         │   │  devfreq-cooling         │ │
  │  │  Battery thermal zone     │   │  Fan cooling             │ │
  │  │  Board/PCB thermal zone   │   │  Peltier cooling         │ │
  │  │                           │   │  Custom cooling           │ │
  │  │  Each has:               │   │                           │ │
  │  │  - Temperature sensor     │   │  Each has:               │ │
  │  │  - Trip points           │   │  - cur_state              │ │
  │  │  - Bound cooling devices │   │  - max_state              │ │
  │  └──────────┬───────────────┘   └──────────┬───────────────┘ │
  │             │                               │                 │
  │  ┌──────────▼───────────────────────────────▼───────────────┐ │
  │  │              Thermal Core (drivers/thermal/)              │ │
  │  │                                                           │ │
  │  │  Polling: reads temperatures periodically                │ │
  │  │  Governor: decides throttling action                     │ │
  │  │  Notification: sends events to userspace                 │ │
  │  └──────────────────────┬───────────────────────────────────┘ │
  │                         │                                     │
  │  ┌──────────────────────▼───────────────────────────────────┐ │
  │  │              Thermal Governors                            │ │
  │  │                                                           │ │
  │  │  step_wise       - Gradual throttle by trip level        │ │
  │  │  power_allocator - PID-based power budget (IPA)          │ │
  │  │  bang_bang        - On/Off for fans                       │ │
  │  │  user_space       - Delegate to userspace daemon          │ │
  │  └──────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. Thermal Zone

```c
/* Registering a thermal zone */
#include <linux/thermal.h>

static int my_get_temp(struct thermal_zone_device *tz, int *temp)
{
    struct my_sensor *sensor = thermal_zone_device_priv(tz);

    /* Read temperature from hardware (millidegree Celsius) */
    *temp = readl(sensor->base + TEMP_REG) * 1000;
    /* e.g., 45°C → temp = 45000 */

    return 0;
}

static const struct thermal_zone_device_ops my_tz_ops = {
    .get_temp = my_get_temp,
};

static int my_sensor_probe(struct platform_device *pdev)
{
    struct thermal_zone_device *tz;

    tz = devm_thermal_of_zone_register(
        &pdev->dev,
        0,          /* sensor ID */
        sensor,     /* driver data */
        &my_tz_ops
    );

    return PTR_ERR_OR_ZERO(tz);
}
```

---

## 3. Trip Points

```
  Trip Points for CPU Thermal Zone (example)
  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  Temperature (°C)                                     │
  │   │                                                   │
  │  100 ─ ─ ─ ─ ─ ─ ─ ─ CRITICAL ─── System shutdown  │
  │   │                    (trip 3)                       │
  │  95 ─ ─ ─ ─ ─ ─ ─ ─  HOT      ─── Emergency action │
  │   │                    (trip 2)                       │
  │  85 ─ ─ ─ ─ ─ ─ ─ ─  PASSIVE  ─── Throttle CPU     │
  │   │                    (trip 1)      (reduce freq)    │
  │  70 ─ ─ ─ ─ ─ ─ ─ ─  ACTIVE   ─── Turn on fan      │
  │   │                    (trip 0)                       │
  │  45 ─                  Normal operation               │
  │   │                                                   │
  │   └──────────────────────────────  Time →             │
  │                                                       │
  │  Trip types:                                          │
  │    ACTIVE   - Activate active cooling (fan)           │
  │    PASSIVE  - Activate passive cooling (throttle)     │
  │    HOT      - Platform-specific emergency actions     │
  │    CRITICAL - Orderly system shutdown                 │
  └──────────────────────────────────────────────────────┘
```

### 3.1 Device Tree Trip Points

```dts
cpu_thermal: thermal-zone {
    polling-delay-passive = <250>;   /* 250ms when throttling */
    polling-delay = <1000>;          /* 1s when idle */

    thermal-sensors = <&tsensor 0>;

    trips {
        cpu_active: active-trip {
            temperature = <70000>;   /* 70°C in millidegrees */
            hysteresis  = <2000>;    /* 2°C hysteresis */
            type = "active";
        };

        cpu_passive: passive-trip {
            temperature = <85000>;   /* 85°C */
            hysteresis  = <5000>;    /* 5°C */
            type = "passive";
        };

        cpu_hot: hot-trip {
            temperature = <95000>;   /* 95°C */
            hysteresis  = <5000>;
            type = "hot";
        };

        cpu_critical: critical-trip {
            temperature = <100000>;  /* 100°C */
            hysteresis  = <0>;       /* No hysteresis */
            type = "critical";
        };
    };

    cooling-maps {
        fan-cooling {
            trip = <&cpu_active>;
            cooling-device = <&fan0 THERMAL_NO_LIMIT
                               THERMAL_NO_LIMIT>;
        };

        cpu-throttle {
            trip = <&cpu_passive>;
            cooling-device = <&cpu0 0 5>;
            /* states 0 to 5 (freq reduction levels) */
        };
    };
};
```

---

## 4. Cooling Devices

### 4.1 CPU Frequency Cooling

```c
/* CPUFreq cooling device registration */
#include <linux/cpu_cooling.h>

static int cpu_thermal_probe(struct platform_device *pdev)
{
    struct thermal_cooling_device *cdev;

    /* Register CPUFreq as cooling device */
    cdev = of_cpufreq_cooling_register(policy);

    /*
     * Cooling states map to frequency levels:
     *
     * State 0: No throttling (max freq)
     *   → CPU runs at 2.0 GHz
     * State 1: 1st reduction
     *   → CPU limited to 1.8 GHz
     * State 2: 2nd reduction
     *   → CPU limited to 1.5 GHz
     * State 3: 3rd reduction
     *   → CPU limited to 1.2 GHz
     * ...
     * State N: Maximum throttle (min freq)
     *   → CPU limited to 500 MHz
     */

    return 0;
}
```

### 4.2 Fan Cooling

```c
/* Fan cooling device driver */
static int fan_get_max_state(struct thermal_cooling_device *cdev,
                              unsigned long *state)
{
    *state = 3; /* Off, Low, Medium, High */
    return 0;
}

static int fan_get_cur_state(struct thermal_cooling_device *cdev,
                              unsigned long *state)
{
    struct fan_data *fan = cdev->devdata;
    *state = fan->current_speed;
    return 0;
}

static int fan_set_cur_state(struct thermal_cooling_device *cdev,
                              unsigned long state)
{
    struct fan_data *fan = cdev->devdata;

    switch (state) {
    case 0: set_pwm(fan, 0);    break; /* Off */
    case 1: set_pwm(fan, 80);   break; /* Low: ~30% */
    case 2: set_pwm(fan, 160);  break; /* Med: ~63% */
    case 3: set_pwm(fan, 255);  break; /* High: 100% */
    }

    fan->current_speed = state;
    return 0;
}

static const struct thermal_cooling_device_ops fan_cooling_ops = {
    .get_max_state = fan_get_max_state,
    .get_cur_state = fan_get_cur_state,
    .set_cur_state = fan_set_cur_state,
};

static int fan_probe(struct platform_device *pdev)
{
    thermal_of_cooling_device_register(
        pdev->dev.of_node,       /* DT node */
        "fan",                   /* type string */
        fan,                     /* driver data */
        &fan_cooling_ops
    );
}
```

---

## 5. Thermal Governors

### 5.1 Step-Wise Governor

```
  step_wise Governor Operation
  ┌──────────────────────────────────────────────────┐
  │                                                   │
  │  Algorithm: Increase/decrease cooling state by 1 │
  │  at each polling interval based on temperature    │
  │                                                   │
  │  Temperature  │  Trend     │  Action              │
  │  ─────────────┼────────────┼──────────────────────│
  │  > trip       │  Rising    │  state += 1          │
  │  > trip       │  Stable    │  (no change)         │
  │  > trip       │  Dropping  │  (no change)         │
  │  < trip-hyst  │  Any       │  state -= 1          │
  │                                                   │
  │  Example with 85°C passive trip, 5°C hysteresis: │
  │                                                   │
  │  T=86°C, rising  → throttle to state 1           │
  │  T=87°C, rising  → throttle to state 2           │
  │  T=86°C, dropping → stay at state 2              │
  │  T=82°C, dropping → still >80 (85-5), stay       │
  │  T=79°C, dropping → below 80°C, go to state 1   │
  │  T=78°C, dropping → state 0 (unthrottled)        │
  └──────────────────────────────────────────────────┘
```

### 5.2 Power Allocator Governor (IPA)

```
  Intelligent Power Allocation (IPA)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Goal: Maximize performance within thermal budget         │
  │                                                           │
  │  PID Controller:                                         │
  │  ┌────────────────────────────────────────────────┐      │
  │  │                                                │      │
  │  │  T_desired (passive trip) ──────┐              │      │
  │  │                                 ├─►[PID]──► P_total │ │
  │  │  T_current (sensor reading) ───┘              │      │
  │  │                                                │      │
  │  │  P_total = Σ(granted power to each actor)      │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Power distribution to actors:                           │
  │  Actor = cooling device with power model                 │
  │                                                           │
  │    CPU: requested 4000mW, weight 1024                    │
  │    GPU: requested 3000mW, weight 1024                    │
  │                                                           │
  │  If P_total = 5000mW (thermal budget):                   │
  │    CPU gets: 5000 × 4000/(4000+3000) = 2857mW           │
  │    GPU gets: 5000 × 3000/(4000+3000) = 2143mW           │
  │                                                           │
  │  Each actor maps granted power → cooling state           │
  │  (lower power = higher cooling state = more throttled)   │
  └──────────────────────────────────────────────────────────┘
```

```dts
/* IPA configuration in device tree */
cpu_thermal: thermal-zone {
    polling-delay-passive = <100>;
    polling-delay = <1000>;

    /* Sustainable power in milliwatts */
    sustainable-power = <4500>;

    /* PID coefficients (optional, auto-tuned if absent) */
    coefficients = <0 250>;
    /* k_pu=0 (unused), k_po=250 (proportional) */

    trips {
        cpu_switch_on: switch-on {
            temperature = <75000>;
            hysteresis = <2000>;
            type = "passive";
        };

        cpu_control: control-temp {
            temperature = <85000>;
            hysteresis = <2000>;
            type = "passive";
        };
    };

    cooling-maps {
        cpu-ipa {
            trip = <&cpu_control>;
            cooling-device = <&cpu0 0 10>;
            contribution = <1024>; /* weight */
        };
        gpu-ipa {
            trip = <&cpu_control>;
            cooling-device = <&gpu 0 5>;
            contribution = <512>;  /* half weight of CPU */
        };
    };
};
```

### 5.3 Bang-Bang Governor

```c
/*
 * Simple on/off control — ideal for fans
 *
 * If temperature >= trip point:
 *     cooling state = max_state  (fan ON full)
 * If temperature < trip point - hysteresis:
 *     cooling state = 0          (fan OFF)
 *
 * No intermediate states — binary control
 */
```

---

## 6. Thermal Monitoring and Debugging

```bash
# List all thermal zones
ls /sys/class/thermal/
# thermal_zone0  thermal_zone1  cooling_device0  cooling_device1

# Read CPU temperature
cat /sys/class/thermal/thermal_zone0/type    # cpu-thermal
cat /sys/class/thermal/thermal_zone0/temp    # 45000 (45°C)

# Trip points
cat /sys/class/thermal/thermal_zone0/trip_point_0_temp  # 70000
cat /sys/class/thermal/thermal_zone0/trip_point_0_type  # active
cat /sys/class/thermal/thermal_zone0/trip_point_1_temp  # 85000
cat /sys/class/thermal/thermal_zone0/trip_point_1_type  # passive

# Current governor
cat /sys/class/thermal/thermal_zone0/policy     # step_wise
# Change governor
echo power_allocator > /sys/class/thermal/thermal_zone0/policy

# Cooling device state
cat /sys/class/thermal/cooling_device0/type       # cpufreq
cat /sys/class/thermal/cooling_device0/cur_state   # 0
cat /sys/class/thermal/cooling_device0/max_state   # 10

# IPA statistics (when using power_allocator)
cat /sys/class/thermal/thermal_zone0/sustainable_power

# Trace thermal events
echo 1 > /sys/kernel/debug/tracing/events/thermal/enable
cat /sys/kernel/debug/tracing/trace_pipe
# thermal_temperature: thermal_zone=cpu-thermal temp=85500
# thermal_power_allocator: ...granted_power=3500
```

---

## 7. Thermal Throttling Flow

```
  Thermal Event Flow
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  [Polling Timer fires]                                   │
  │       │                                                   │
  │       ▼                                                   │
  │  [tz->ops->get_temp()]  ← Read sensor hardware           │
  │       │                                                   │
  │       ▼                                                   │
  │  [thermal_zone_device_update()]                          │
  │       │                                                   │
  │       ├─► Check trip points (low → high)                 │
  │       │   │                                               │
  │       │   ├─ T < all trips → No action                   │
  │       │   ├─ T ≥ active trip → Notify governor           │
  │       │   ├─ T ≥ passive trip → Notify governor          │
  │       │   ├─ T ≥ hot trip → thermal_emergency_poweroff() │
  │       │   └─ T ≥ critical trip → orderly_poweroff()      │
  │       │                                                   │
  │       ▼                                                   │
  │  [Governor decides cooling state]                        │
  │       │                                                   │
  │       ▼                                                   │
  │  [cdev->ops->set_cur_state()] ← Apply throttling        │
  │       │                                                   │
  │       ├─ CPUFreq cooling: limit max frequency            │
  │       ├─ DevFreq cooling: limit GPU/bus frequency        │
  │       └─ Fan cooling: set PWM duty cycle                 │
  │                                                           │
  │  [Switch to passive polling interval if throttling]      │
  └──────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/thermal/thermal_core.c` | Thermal framework core |
| `drivers/thermal/thermal_of.c` | DT thermal zone parsing |
| `drivers/thermal/gov_step_wise.c` | Step-wise governor |
| `drivers/thermal/gov_power_allocator.c` | IPA governor |
| `drivers/thermal/gov_bang_bang.c` | Bang-bang governor |
| `drivers/thermal/cpufreq_cooling.c` | CPU frequency cooling |
| `drivers/thermal/devfreq_cooling.c` | Device frequency cooling |
| `include/linux/thermal.h` | Thermal framework API |
| `include/dt-bindings/thermal/thermal.h` | DT thermal constants |

---

## Interview Questions

**Q1: Explain the difference between ACTIVE and PASSIVE trip points.**
**A:** ACTIVE trip points trigger active cooling — mechanisms that consume power to remove heat, like fans (forced convection). PASSIVE trip points trigger passive cooling — reducing heat generation by throttling performance (lowering CPU/GPU frequency). ACTIVE cooling maintains performance but requires power and mechanical components. PASSIVE cooling trades performance for thermal headroom, is silent, and has no moving parts. Mobile devices primarily use passive cooling; servers/desktops combine both.

**Q2: How does the power_allocator (IPA) governor distribute power?**
**A:** IPA uses a PID controller with target temperature = passive trip point. It calculates total power budget (P_total) from the PID output. Each cooling device (actor) requests power based on its current workload. IPA distributes P_total proportionally to each actor's requested power, weighted by the `contribution` value from DT. Actors then map their granted power to a cooling state — less granted power means higher cooling state (more throttled). This maximizes overall system performance within the thermal envelope.

**Q3: What is hysteresis and why is it needed?**
**A:** Hysteresis prevents oscillation at trip point boundaries. Without hysteresis, if a trip point is 85°C, temperature fluctuating between 84.9°C and 85.1°C would cause rapid on/off throttling — degrading performance and user experience. With 5°C hysteresis, throttling activates at 85°C but only deactivates when temperature drops below 80°C (85-5). This creates a stable dead zone preventing flip-flopping.

---

## Summary

- Thermal framework connects thermal zones (temperature sensors), trip points (thresholds), and cooling devices (throttle actuators)
- Trip types: ACTIVE (fan), PASSIVE (throttle), HOT (emergency), CRITICAL (shutdown)
- Governors: step_wise (gradual), power_allocator (IPA with PID), bang_bang (on/off for fans)
- IPA distributes power budget proportionally to cooling device actors for maximum performance
- Hysteresis prevents oscillation at trip point boundaries
- CPUFreq cooling: maps cooling states to frequency reduction levels
- Monitor via `/sys/class/thermal/` and ftrace thermal events
- DT configures sensors, trips, cooling maps, and governor parameters

---

[Previous Chapter: Regulator Framework ←](Chapter_14_Regulator_Framework.md) | [Next Chapter: Power Management in Device Drivers →](Chapter_16_Driver_PM.md)
