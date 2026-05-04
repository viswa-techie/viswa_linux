# Chapter 12: Power Domains

## Learning Goals
- Understand power domain concepts and SoC power partitioning
- Learn the Generic Power Domain (genpd) framework
- Know how devices are associated with power domains
- Understand domain hierarchy and state management
- Learn power domain configuration via device tree

---

## 1. Power Domain Concepts

```
  Power Domain: A group of hardware blocks sharing a
  common power switch that can be turned on/off together.
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  SoC with Power Domains                                  │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │  VDD_main                                          │  │
  │  │  ┌── PD_always_on ───────────────────┐             │  │
  │  │  │  PMU, RTC, SRAM, Wakeup ctrl      │             │  │
  │  │  │  (NEVER powered off)              │             │  │
  │  │  └───────────────────────────────────┘             │  │
  │  │                                                     │  │
  │  │  ┌── PD_cpu ──────┐  ┌── PD_gpu ──────────┐       │  │
  │  │  │  Power switch  │  │  Power switch       │       │  │
  │  │  │  ┌────┐┌────┐  │  │  ┌─────────────┐   │       │  │
  │  │  │  │CPU0││CPU1│  │  │  │ GPU cores   │   │       │  │
  │  │  │  └────┘└────┘  │  │  │ + L2 cache  │   │       │  │
  │  │  │  ┌──────────┐  │  │  └─────────────┘   │       │  │
  │  │  │  │ L2 cache │  │  │                     │       │  │
  │  │  │  └──────────┘  │  │  Can be OFF when   │       │  │
  │  │  └────────────────┘  │  no GPU workload    │       │  │
  │  │                      └─────────────────────┘       │  │
  │  │                                                     │  │
  │  │  ┌── PD_display ──┐  ┌── PD_camera ────────┐      │  │
  │  │  │  Display ctrl  │  │  ISP + CSI PHY      │      │  │
  │  │  │  MIPI DSI      │  │                     │      │  │
  │  │  └────────────────┘  └─────────────────────┘      │  │
  │  │                                                     │  │
  │  │  ┌── PD_periph ──────────────────────────────┐     │  │
  │  │  │  UART, I2C, SPI, USB controllers          │     │  │
  │  │  └───────────────────────────────────────────┘     │  │
  │  └────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Generic Power Domain (genpd) Framework

### 2.1 Architecture

```
  genpd Framework Architecture
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── genpd Core (drivers/base/power/domain.c) ────────┐ │
  │  │                                                      │ │
  │  │  struct generic_pm_domain {                         │ │
  │  │      char *name;                                    │ │
  │  │      struct list_head master_links;  /* parents */  │ │
  │  │      struct list_head slave_links;   /* children */ │ │
  │  │      struct list_head dev_list;      /* devices */  │ │
  │  │      enum gpd_status status;         /* on/off */   │ │
  │  │      int (*power_on)(struct generic_pm_domain *);   │ │
  │  │      int (*power_off)(struct generic_pm_domain *);  │ │
  │  │      /* ... */                                      │ │
  │  │  };                                                 │ │
  │  │                                                      │ │
  │  │  Key operations:                                     │ │
  │  │  - pm_genpd_init()     — Register domain            │ │
  │  │  - pm_genpd_add_device() — Associate device         │ │
  │  │  - pm_genpd_add_subdomain() — Create hierarchy      │ │
  │  └──────────────────────────────────────────────────────┘ │
  │                                                           │
  │  Integration:                                            │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │  Runtime PM ←→ genpd:                            │    │
  │  │  When ALL devices in domain runtime-suspend      │    │
  │  │  → genpd calls power_off()                       │    │
  │  │  When ANY device needs runtime-resume             │    │
  │  │  → genpd calls power_on() first                  │    │
  │  └──────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

### 2.2 genpd Provider Implementation

```c
/* SoC-specific power domain driver */
#include <linux/pm_domain.h>

struct my_soc_pd {
    struct generic_pm_domain genpd;
    void __iomem *pmu_regs;
    unsigned int pd_id;
};

static int my_power_on(struct generic_pm_domain *domain)
{
    struct my_soc_pd *pd = container_of(domain, struct my_soc_pd, genpd);

    /* Power-up sequence:
     * 1. Assert isolation
     * 2. Turn on power switch
     * 3. Wait for power good
     * 4. De-assert isolation
     * 5. De-assert reset
     */
    writel(PD_POWER_ON, pd->pmu_regs + PD_CTRL(pd->pd_id));

    /* Wait for power-good signal */
    return readl_poll_timeout(pd->pmu_regs + PD_STATUS(pd->pd_id),
                              val, val & PD_POWERED, 10, 1000);
}

static int my_power_off(struct generic_pm_domain *domain)
{
    struct my_soc_pd *pd = container_of(domain, struct my_soc_pd, genpd);

    /* Power-down sequence:
     * 1. Assert reset
     * 2. Assert isolation
     * 3. Turn off power switch
     */
    writel(PD_POWER_OFF, pd->pmu_regs + PD_CTRL(pd->pd_id));
    return 0;
}

static int my_soc_pd_probe(struct platform_device *pdev)
{
    struct my_soc_pd *pd;

    pd = devm_kzalloc(&pdev->dev, sizeof(*pd), GFP_KERNEL);

    pd->genpd.name = "pd_gpu";
    pd->genpd.power_on = my_power_on;
    pd->genpd.power_off = my_power_off;

    /* Register the power domain */
    pm_genpd_init(&pd->genpd, NULL, true); /* start powered off */

    /* Register as OF provider for device tree */
    of_genpd_add_provider_simple(pdev->dev.of_node, &pd->genpd);

    return 0;
}
```

### 2.3 Device Tree Configuration

```dts
/* Power domain provider in device tree */
power-controller@50000 {
    compatible = "my-soc,power-controller";
    reg = <0x50000 0x1000>;
    #power-domain-cells = <1>;

    pd_gpu: power-domain@0 {
        reg = <0>;
        #power-domain-cells = <0>;
    };

    pd_display: power-domain@1 {
        reg = <1>;
        #power-domain-cells = <0>;
    };

    pd_camera: power-domain@2 {
        reg = <2>;
        #power-domain-cells = <0>;
        /* Subdomain of pd_display */
        power-domains = <&pd_display>;
    };
};

/* Consumer device associated with power domain */
gpu@10000 {
    compatible = "my-soc,gpu";
    reg = <0x10000 0x1000>;
    power-domains = <&pd_gpu>;
    /* genpd automatically manages power on get/put */
};

display@20000 {
    compatible = "my-soc,display";
    reg = <0x20000 0x1000>;
    power-domains = <&pd_display>;
};
```

---

## 3. Domain Hierarchy

```
  Power Domain Hierarchy Rules
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Parent-child domain relationships:                      │
  │                                                           │
  │  ┌── PD_soc (root) ──────────────────────────────────┐  │
  │  │                                                     │  │
  │  │  ┌── PD_multimedia ──────────────────────────────┐ │  │
  │  │  │                                               │ │  │
  │  │  │  ┌── PD_display ──┐  ┌── PD_camera ────────┐│ │  │
  │  │  │  │  Display ctrl  │  │  ISP, CSI            ││ │  │
  │  │  │  └────────────────┘  └──────────────────────┘│ │  │
  │  │  └───────────────────────────────────────────────┘ │  │
  │  │                                                     │  │
  │  │  ┌── PD_connectivity ────────────────────────────┐ │  │
  │  │  │  ┌── PD_wifi ──┐  ┌── PD_bluetooth ────────┐│ │  │
  │  │  │  │  WiFi HW    │  │  BT HW                 ││ │  │
  │  │  │  └─────────────┘  └────────────────────────┘│ │  │
  │  │  └───────────────────────────────────────────────┘ │  │
  │  └─────────────────────────────────────────────────────┘  │
  │                                                           │
  │  Rules:                                                  │
  │  1. Parent ON if ANY child is ON                         │
  │  2. Parent OFF only when ALL children are OFF            │
  │  3. Child cannot be ON if parent is OFF                  │
  │  4. Power-on: parent first, then child                   │
  │  5. Power-off: child first, then parent                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Domain States

```c
/* genpd performance states (for DVFS within domains) */

/* A power domain can have multiple performance states */
/* Each state = different voltage/frequency for the domain */

/* OPP table per domain */
static struct genpd_power_state pd_states[] = {
    {
        .power_off_latency_ns = 5000,    /* 5μs */
        .power_on_latency_ns  = 10000,   /* 10μs */
        .residency_ns         = 50000,   /* 50μs min */
    },
    {
        .power_off_latency_ns = 100000,  /* 100μs */
        .power_on_latency_ns  = 200000,  /* 200μs */
        .residency_ns         = 500000,  /* 500μs min */
    },
};

/* Domain with multiple idle states */
pd->genpd.states = pd_states;
pd->genpd.state_count = ARRAY_SIZE(pd_states);
```

---

## 5. genpd and Runtime PM Integration

```
  genpd + Runtime PM Flow
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Device A (in PD_gpu) does pm_runtime_put():             │
  │  1. Device A's runtime_suspend callback runs             │
  │  2. genpd checks: any other active devices in domain?   │
  │     ├── Yes → domain stays ON                            │
  │     └── No  → genpd calls pd->power_off()                │
  │              → Power switch turns off PD_gpu              │
  │              → All hardware in domain loses power         │
  │                                                           │
  │  Device B (in PD_gpu) does pm_runtime_get_sync():        │
  │  1. genpd checks: is domain ON?                          │
  │     ├── Yes → skip to step 3                             │
  │     └── No  → genpd calls pd->power_on()                 │
  │              → Power switch turns on PD_gpu               │
  │              → Wait for power stabilization               │
  │  2. Domain is now powered                                │
  │  3. Device B's runtime_resume callback runs              │
  │                                                           │
  │  Key: Drivers don't need to know about power domains!    │
  │  They just use standard runtime PM get/put.              │
  │  genpd handles domain power transitions transparently.   │
  └──────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/base/power/domain.c` | Generic power domain core |
| `drivers/base/power/domain_governor.c` | genpd governor (decides domain state) |
| `include/linux/pm_domain.h` | genpd API and structures |
| `drivers/soc/*/pm_domains.c` | SoC-specific domain implementations |
| `Documentation/power/power-domain.rst` | Power domain documentation |

---

## Interview Questions

**Q1: What is a power domain and why is it important?**
**A:** A power domain is a group of hardware blocks sharing a common power switch. When all devices in a domain are idle, the entire domain can be powered off, eliminating both dynamic and leakage power. This is especially important in SoCs where dozens of hardware blocks (GPU, camera, display) are inactive most of the time. Per-domain gating can save 100s of mW per domain in an idle state.

**Q2: How does genpd integrate with runtime PM?**
**A:** genpd transparently intercepts runtime PM callbacks. When a device does pm_runtime_put() and is the last active device in its domain, genpd calls its power_off() callback to gate the domain. When any device does pm_runtime_get_sync(), genpd first ensures the domain is powered (calling power_on() if needed) before running the device's runtime_resume. Device drivers use standard runtime PM APIs — they don't need to be aware of power domains.

**Q3: Explain genpd parent-child relationships.**
**A:** Power domains can form hierarchies: a parent domain must be ON if any child domain is ON (because the child's power often derives from the parent). When all children are OFF, the parent can be powered off too. Power-on order is parent-first, power-off order is child-first. Example: PD_multimedia contains PD_display and PD_camera. If only the camera is active, both PD_camera and PD_multimedia are on; PD_display can be off.

---

## Summary

- Power domains group hardware blocks that share a power switch for collective on/off
- genpd framework provides a generic interface for SoC power domain management
- Domain hierarchy enforces parent-ON-when-child-ON relationships
- genpd integrates transparently with runtime PM — drivers use standard get/put
- Device tree associates devices with power domains via `power-domains` property
- Domain states support multiple idle levels with varying latency/residency
- SoC-specific drivers implement power_on/power_off callbacks for actual hardware control

---

[Previous Chapter: Suspend and Resume ←](Chapter_11_Suspend_Resume.md) | [Next Chapter: Clock Framework →](Chapter_13_Clock_Framework.md)
