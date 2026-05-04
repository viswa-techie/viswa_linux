# Chapter 16: Thermal Framework

## Learning Goals
- Understand Linux thermal management architecture
- Know thermal zones, cooling devices, and governors
- Implement thermal sensor and cooling device drivers
- Configure thermal policies via device tree

---

## 16.1 Thermal Framework Architecture

```
Thermal Framework:

  ┌──────────────────────────────────────────────────┐
  │  Thermal Core (drivers/thermal/thermal_core.c)    │
  │                                                   │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  │
  │  │  Thermal   │  │  Cooling   │  │  Governor  │  │
  │  │  Zone      │  │  Device    │  │            │  │
  │  │  (sensor)  │  │  (actuator)│  │  (policy)  │  │
  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  │
  │        │               │               │          │
  │  Maps sensors to cooling devices with trip points │
  └──────────────────────────────────────────────────┘

Thermal Zone:
  ├── Temperature sensor (reads current temp)
  ├── Trip points (thresholds for action)
  │   ├── passive (80°C) → throttle performance
  │   ├── active  (70°C) → turn on fan
  │   └── critical(120°C)→ emergency shutdown
  └── Bound cooling devices (what to do at each trip)

Cooling Device:
  ├── CPU frequency scaling (cpufreq)
  ├── GPU frequency scaling
  ├── Fan controller (PWM)
  ├── Battery charging current limit
  └── Device-specific throttling

Governor (policy):
  ├── step_wise    → increment/decrement cooling state at trips
  ├── bang_bang    → on/off at trip (for fans)
  ├── power_allocator → PID-based power budget distribution
  └── user_space   → delegate to user-space daemon
```

---

## 16.2 Thermal Control Loop

```
Thermal Control Loop:

                    ┌─────────────┐
                    │ Temperature │
                    │   Sensor    │
                    │  (read temp)│
                    └──────┬──────┘
                           │ 85°C
                    ┌──────▼──────┐
                    │  Thermal    │
                    │  Zone       │
                    │  Compare    │
                    │  to trips   │
                    └──────┬──────┘
                           │ > passive trip (80°C)
                    ┌──────▼──────┐
                    │  Governor   │
                    │  (step_wise)│
                    │  Decide     │
                    │  action     │
                    └──────┬──────┘
                           │ increase cooling
                    ┌──────▼──────┐
                    │  Cooling    │
                    │  Device     │
                    │  (cpufreq)  │
                    │  Reduce     │
                    │  frequency  │
                    └─────────────┘
                    2.0GHz → 1.5GHz

Temperature vs Action:
  Temp (°C)  │ Action
  ───────────┼──────────────────────────
    < 70     │ No throttling (full speed)
   70 - 80   │ Fan ON (active cooling)
   80 - 100  │ CPU/GPU frequency reduced
   100 - 110 │ Heavy throttling
    > 120    │ CRITICAL: emergency shutdown
```

---

## 16.3 Thermal Zone Driver (Sensor)

```c
/* Thermal zone — temperature sensor driver */
#include <linux/thermal.h>

static int my_sensor_get_temp(struct thermal_zone_device *tz, int *temp)
{
    struct my_sensor *priv = thermal_zone_device_priv(tz);

    /* Read temperature from hardware (millidegrees Celsius) */
    u32 raw = readl(priv->regs + TEMP_DATA_REG);
    *temp = (raw * 1000) - 40000;  /* Convert to millidegrees */

    return 0;
}

static const struct thermal_zone_device_ops my_tz_ops = {
    .get_temp = my_sensor_get_temp,
};

static int my_sensor_probe(struct platform_device *pdev)
{
    struct my_sensor *priv;
    struct thermal_zone_device *tz;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    priv->regs = devm_platform_ioremap_resource(pdev, 0);

    /* Register thermal zone (polling every 1000ms) */
    tz = devm_thermal_of_zone_register(&pdev->dev, 0,
                                        priv, &my_tz_ops);
    if (IS_ERR(tz))
        return PTR_ERR(tz);

    return 0;
}
```

---

## 16.4 Cooling Device Driver

```c
/* Cooling device — for custom cooling (e.g., fan, device throttle) */

static int my_fan_get_max_state(struct thermal_cooling_device *cdev,
                                 unsigned long *max_state)
{
    *max_state = 3;  /* 4 levels: 0=off, 1=low, 2=med, 3=high */
    return 0;
}

static int my_fan_get_cur_state(struct thermal_cooling_device *cdev,
                                 unsigned long *state)
{
    struct my_fan *priv = cdev->devdata;
    *state = priv->current_speed;
    return 0;
}

static int my_fan_set_cur_state(struct thermal_cooling_device *cdev,
                                 unsigned long state)
{
    struct my_fan *priv = cdev->devdata;
    /* PWM duty cycle: 0=off, 1=33%, 2=66%, 3=100% */
    unsigned int duty = (state * 255) / 3;
    pwm_config(priv->pwm, duty, 255);
    priv->current_speed = state;
    return 0;
}

static const struct thermal_cooling_device_ops my_fan_ops = {
    .get_max_state = my_fan_get_max_state,
    .get_cur_state = my_fan_get_cur_state,
    .set_cur_state = my_fan_set_cur_state,
};

static int my_fan_probe(struct platform_device *pdev)
{
    struct thermal_cooling_device *cdev;

    cdev = devm_thermal_of_cooling_device_register(&pdev->dev,
                pdev->dev.of_node, "my-fan", priv, &my_fan_ops);
    return PTR_ERR_OR_ZERO(cdev);
}
```

---

## 16.5 Device Tree Thermal Configuration

```dts
/* Thermal zones in device tree */
thermal-zones {
    cpu-thermal {
        polling-delay-passive = <250>;  /* ms (when throttling) */
        polling-delay = <1000>;         /* ms (when idle) */
        thermal-sensors = <&tsens 0>;   /* sensor reference */

        trips {
            cpu_active: active-trip {
                temperature = <70000>;  /* 70°C (millidegrees) */
                hysteresis  = <2000>;   /* 2°C hysteresis */
                type = "active";
            };

            cpu_passive: passive-trip {
                temperature = <80000>;  /* 80°C */
                hysteresis  = <5000>;   /* 5°C */
                type = "passive";
            };

            cpu_critical: critical-trip {
                temperature = <120000>; /* 120°C */
                hysteresis  = <0>;
                type = "critical";      /* emergency shutdown */
            };
        };

        cooling-maps {
            /* At active-trip: turn on fan */
            map0 {
                trip = <&cpu_active>;
                cooling-device = <&fan 0 3>;  /* min=0, max=3 */
            };

            /* At passive-trip: throttle CPU */
            map1 {
                trip = <&cpu_passive>;
                cooling-device = <&cpu0 0 4>;  /* cpufreq levels */
            };
        };
    };
};
```

---

## 16.6 Debug

```bash
# List thermal zones
$ ls /sys/class/thermal/
$ cat /sys/class/thermal/thermal_zone0/type
$ cat /sys/class/thermal/thermal_zone0/temp

# Trip points
$ cat /sys/class/thermal/thermal_zone0/trip_point_0_temp
$ cat /sys/class/thermal/thermal_zone0/trip_point_0_type

# Cooling devices
$ cat /sys/class/thermal/cooling_device0/type
$ cat /sys/class/thermal/cooling_device0/cur_state
$ cat /sys/class/thermal/cooling_device0/max_state

# Policy/governor
$ cat /sys/class/thermal/thermal_zone0/policy
$ echo step_wise > /sys/class/thermal/thermal_zone0/policy

# Trace thermal events
$ trace-cmd record -e thermal
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| Thermal core | drivers/thermal/thermal_core.c | Framework core |
| Governors | drivers/thermal/gov_*.c | Policy engines |
| thermal.h | include/linux/thermal.h | API definitions |
| cpufreq cooling | drivers/thermal/cpufreq_cooling.c | CPU frequency throttle |
| DT bindings | drivers/thermal/thermal_of.c | Device tree parsing |

---

## Interview Questions

**Q1: Explain the thermal framework components and their interaction.**
A: Three components: (1) **Thermal zone** — represents a thermal sensor, provides `get_temp()` callback. Defines trip points (temperature thresholds) and polling interval. (2) **Cooling device** — an actuator that reduces heat (CPU frequency reduction, fan speed increase, device shutdown). Has configurable cooling states (0=min, max=full cooling). (3) **Governor** — policy engine that decides how much cooling to apply based on temperature relative to trip points. `step_wise` increases/decreases cooling state one step at a time; `power_allocator` uses PID control. Cooling-maps in device tree bind specific cooling devices to specific trip points.

**Q2: What is the difference between active, passive, and critical trip points?**
A: **Active**: Typically used for active cooling (e.g., turning on a fan). The governor activates the bound cooling device when temperature exceeds this threshold. **Passive**: Used for device throttling — reduces performance to lower heat (CPU frequency reduction, GPU clock lowering). The polling interval switches to `polling-delay-passive` (faster) for tighter control. **Critical**: Emergency — system must shut down immediately to prevent hardware damage. The thermal core triggers `orderly_poweroff()` when this trip is reached. No cooling device can mitigate it.

**Q3: How does hysteresis work in thermal management?**
A: Hysteresis prevents oscillation at trip boundaries. If a passive trip is at 80°C with 5°C hysteresis, the cooling device activates at 80°C but deactivates only when temperature drops to 75°C. Without hysteresis, the system would rapidly toggle between throttled and unthrottled states as temperature oscillates around 80°C — causing performance instability and potentially faster thermal cycling damage.

---

## Summary

- Thermal framework: zones (sensors) + cooling devices (actuators) + governors (policy)
- Trip types: active (fan), passive (throttle), critical (shutdown)
- Governors: step_wise (incremental), bang_bang (on/off), power_allocator (PID)
- Device tree defines thermal zones, trips, and cooling-maps
- Hysteresis prevents oscillation at trip boundaries
- CPU frequency scaling is the most common cooling device in embedded

---

*Next: [Chapter 17 — Device Tree Integration](Chapter_17_Device_Tree.md)*
