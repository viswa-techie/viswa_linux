# Chapter 17: Power Management and Device Tree

## Learning Goals
- Understand how device tree describes power management hardware
- Learn DT bindings for power domains, clocks, regulators, and thermal
- Know operating performance points (OPP) table configuration
- Understand wakeup source and PM property DT bindings
- Learn DT overlays for PM tuning

---

## 1. Device Tree PM Properties Overview

```
  DT Power Management Bindings Map
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  device@addr {                                               │
  │      /* Power Domains */                                     │
  │      power-domains = <&pd_ctrl DOMAIN_ID>;                   │
  │      power-domain-names = "core", "io";                      │
  │                                                               │
  │      /* Clocks */                                            │
  │      clocks = <&clk_ctrl CLK_ID>;                            │
  │      clock-names = "bus", "core";                            │
  │      assigned-clocks = <&clk_ctrl CLK_ID>;                   │
  │      assigned-clock-rates = <100000000>;                     │
  │                                                               │
  │      /* Regulators */                                        │
  │      vdd-supply = <&regulator_handle>;                       │
  │      vdd-io-supply = <&io_regulator>;                        │
  │                                                               │
  │      /* OPP (DVFS) */                                        │
  │      operating-points-v2 = <&opp_table>;                     │
  │                                                               │
  │      /* Wakeup */                                            │
  │      wakeup-source;                                          │
  │                                                               │
  │      /* Interconnect bandwidth */                            │
  │      interconnects = <&noc MASTER_ID &noc SLAVE_ID>;         │
  │      interconnect-names = "dma-mem";                         │
  │  };                                                           │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. Power Domain DT Bindings

```dts
/* Power domain controller */
power: power-controller@10000 {
    compatible = "my-soc,power-controller";
    reg = <0x10000 0x1000>;
    #power-domain-cells = <1>;
    /* 1 cell = domain index number */
};

/* Domain hierarchy (optional) */
pd_gpu: power-domain-gpu {
    #power-domain-cells = <0>;
    power-domains = <&power PD_TOP>;  /* Parent domain */
    domain-idle-states = <&gpu_pd_off>;
};

/* Domain idle states */
gpu_pd_off: domain-idle-state {
    compatible = "domain-idle-state";
    entry-latency-us = <500>;
    exit-latency-us = <800>;
    min-residency-us = <5000>;
};

/* Consumer references power domain */
gpu: gpu@20000 {
    compatible = "my-soc,gpu";
    reg = <0x20000 0x10000>;
    power-domains = <&power PD_GPU>;
    /* genpd framework auto-manages this domain
     * during runtime PM transitions */
};

/* Device with multiple power domains */
isp: isp@30000 {
    compatible = "my-soc,isp";
    power-domains = <&power PD_ISP_CORE>,
                    <&power PD_ISP_IO>;
    power-domain-names = "core", "io";
    /* Requires struct dev_pm_domain_list */
};
```

---

## 3. OPP Table (DVFS Configuration)

```dts
/* Operating Performance Points table */
cpu_opp_table: opp-table {
    compatible = "operating-points-v2";
    opp-shared;  /* All CPUs in cluster share this table */

    opp-500000000 {
        opp-hz = /bits/ 64 <500000000>;   /* 500 MHz */
        opp-microvolt = <800000>;          /* 0.8V */
        opp-supported-hw = <0x3>;          /* HW rev mask */
        clock-latency-ns = <500000>;       /* 500μs transition */
    };

    opp-1000000000 {
        opp-hz = /bits/ 64 <1000000000>;   /* 1.0 GHz */
        opp-microvolt = <1000000>;          /* 1.0V */
        clock-latency-ns = <500000>;
    };

    opp-1500000000 {
        opp-hz = /bits/ 64 <1500000000>;   /* 1.5 GHz */
        opp-microvolt = <1100000>;          /* 1.1V */
        clock-latency-ns = <500000>;
    };

    opp-2000000000 {
        opp-hz = /bits/ 64 <2000000000>;   /* 2.0 GHz */
        opp-microvolt = <1200000>;          /* 1.2V */
        clock-latency-ns = <500000>;
        opp-suspend;                        /* Use during suspend */
    };

    opp-2500000000 {
        opp-hz = /bits/ 64 <2500000000>;   /* 2.5 GHz (turbo) */
        opp-microvolt = <1350000>;          /* 1.35V */
        clock-latency-ns = <500000>;
        turbo-mode;                         /* Only if turbo enabled */
    };
};

cpu@0 {
    compatible = "arm,cortex-a78";
    operating-points-v2 = <&cpu_opp_table>;
    cpu-supply = <&buck1>;       /* Regulator for voltage scaling */
    clocks = <&ccu CLK_CPU>;    /* Clock for frequency scaling */
};
```

### 3.1 OPP with Multiple Supplies

```dts
/* SoC requiring both core and memory voltage */
gpu_opp_table: opp-table-gpu {
    compatible = "operating-points-v2";

    opp-400000000 {
        opp-hz = /bits/ 64 <400000000>;
        opp-microvolt = <900000 850000 950000>,  /* core: nom, min, max */
                        <1100000>;                /* mem voltage */
    };

    opp-800000000 {
        opp-hz = /bits/ 64 <800000000>;
        opp-microvolt = <1100000 1050000 1150000>,
                        <1200000>;
    };
};

gpu: gpu@20000 {
    operating-points-v2 = <&gpu_opp_table>;
    vdd-supply = <&gpu_buck>;     /* Maps to first voltage */
    vdd-mem-supply = <&mem_ldo>;  /* Maps to second voltage */
};
```

---

## 4. Clock DT Bindings for PM

```dts
/* Clock controller */
ccu: clock-controller@1000 {
    compatible = "my-soc,ccu";
    reg = <0x1000 0x400>;
    #clock-cells = <1>;
    clocks = <&osc24m>, <&pll_periph>;
};

/* Device with multiple clocks */
display: display@40000 {
    compatible = "my-soc,display";
    clocks = <&ccu CLK_DISP_BUS>,     /* Bus/AHB clock */
             <&ccu CLK_DISP_CORE>,    /* Pixel clock */
             <&ccu CLK_DISP_PHY>;     /* MIPI PHY clock */
    clock-names = "bus", "core", "phy";

    /* Assign initial clock rates */
    assigned-clocks = <&ccu CLK_DISP_CORE>;
    assigned-clock-rates = <148500000>;   /* 148.5 MHz for 1080p */
    assigned-clock-parents = <&ccu CLK_PLL_VIDEO>;
};
```

---

## 5. Regulator DT Bindings for PM

```dts
/* PMIC with regulator nodes */
pmic: pmic@48 {
    compatible = "my-pmic";
    reg = <0x48>;

    regulators {
        buck1: BUCK1 {
            regulator-name = "vdd-cpu";
            regulator-min-microvolt = <800000>;
            regulator-max-microvolt = <1400000>;
            regulator-always-on;
            regulator-boot-on;
            regulator-ramp-delay = <12500>; /* μV/μs */

            /* Suspend voltage for system sleep */
            regulator-state-mem {
                regulator-on-in-suspend;
                regulator-suspend-microvolt = <800000>;
            };
        };

        buck2: BUCK2 {
            regulator-name = "vdd-gpu";
            regulator-min-microvolt = <600000>;
            regulator-max-microvolt = <1200000>;

            regulator-state-mem {
                regulator-off-in-suspend; /* Turn off GPU in sleep */
            };
        };

        ldo1: LDO1 {
            regulator-name = "vdd-dram";
            regulator-min-microvolt = <1100000>;
            regulator-max-microvolt = <1100000>;
            regulator-always-on;  /* DDR needs power even in sleep */
        };
    };
};
```

---

## 6. Wakeup Source DT Bindings

```dts
/* GPIO key as wakeup source */
gpio-keys {
    compatible = "gpio-keys";

    power-button {
        label = "Power Button";
        gpios = <&gpio 5 GPIO_ACTIVE_LOW>;
        linux,code = <KEY_POWER>;
        wakeup-source;              /* Can wake system */
        wakeup-event-action = <EV_ACT_DEASSERTED>;
    };
};

/* RTC as wakeup source */
rtc: rtc@68 {
    compatible = "nxp,pcf8563";
    reg = <0x68>;
    wakeup-source;
    /* RTC alarm can wake system from S3/S4 */
};

/* UART wakeup (for serial console wake) */
uart0: serial@10000 {
    compatible = "my-soc,uart";
    reg = <0x10000 0x100>;
    interrupts = <GIC_SPI 10 IRQ_TYPE_LEVEL_HIGH>;
    wakeup-source;
    /* Receive data can wake system */
};
```

---

## 7. Thermal DT Bindings

```dts
/* Complete thermal DT configuration */
thermal-zones {
    cpu-thermal {
        polling-delay-passive = <250>;
        polling-delay = <1000>;
        thermal-sensors = <&tsensor 0>;
        sustainable-power = <4500>; /* For IPA governor */

        trips {
            cpu_alert: alert {
                temperature = <75000>;
                hysteresis = <2000>;
                type = "passive";
            };
            cpu_crit: crit {
                temperature = <100000>;
                hysteresis = <0>;
                type = "critical";
            };
        };

        cooling-maps {
            map0 {
                trip = <&cpu_alert>;
                cooling-device = <&cpu0 0 5>;
                contribution = <1024>;
            };
        };
    };
};
```

---

## 8. CPU Idle States in DT

```dts
/* ARM CPU idle states */
cpus {
    #address-cells = <1>;
    #size-cells = <0>;

    cpu0: cpu@0 {
        device_type = "cpu";
        compatible = "arm,cortex-a78";
        enable-method = "psci";
        cpu-idle-states = <&CPU_WFI &CPU_RET &CLUSTER_PD>;
    };

    idle-states {
        entry-method = "psci";

        CPU_WFI: wfi {
            compatible = "arm,idle-state";
            arm,psci-suspend-param = <0x0000000>;
            local-timer-stop;
            entry-latency-us = <1>;
            exit-latency-us = <1>;
            min-residency-us = <10>;
        };

        CPU_RET: retention {
            compatible = "arm,idle-state";
            arm,psci-suspend-param = <0x0010000>;
            local-timer-stop;
            entry-latency-us = <50>;
            exit-latency-us = <100>;
            min-residency-us = <1000>;
        };

        CLUSTER_PD: cluster-power-down {
            compatible = "arm,idle-state";
            arm,psci-suspend-param = <0x1010000>;
            local-timer-stop;
            entry-latency-us = <500>;
            exit-latency-us = <1000>;
            min-residency-us = <5000>;
        };
    };
};

/* PSCI node (firmware interface for idle/hotplug) */
psci {
    compatible = "arm,psci-1.0";
    method = "smc";
};
```

---

## 9. Interconnect Bandwidth DT

```dts
/* Network-on-Chip bandwidth requirements */
noc: interconnect@100000 {
    compatible = "my-soc,noc";
    reg = <0x100000 0x1000>;
    #interconnect-cells = <1>;
};

gpu: gpu@20000 {
    /* Request specific bandwidth for PM */
    interconnects = <&noc MASTER_GPU &noc SLAVE_DDR>;
    interconnect-names = "gpu-mem";
    /* icc framework auto-scales bandwidth during runtime PM */
};
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/opp/of.c` | DT OPP table parsing |
| `drivers/base/power/domain.c` | Power domain DT handling |
| `drivers/clk/clk.c` | Clock DT assigned-clocks |
| `drivers/regulator/of_regulator.c` | Regulator DT parsing |
| `drivers/thermal/thermal_of.c` | Thermal DT parsing |
| `drivers/cpuidle/dt_idle_states.c` | CPU idle DT states |
| `Documentation/devicetree/bindings/power/` | Power DT bindings |
| `Documentation/devicetree/bindings/opp/` | OPP DT bindings |

---

## Interview Questions

**Q1: Explain the OPP table structure in device tree.**
**A:** The OPP (Operating Performance Points) table defines valid frequency-voltage pairs for DVFS. Each entry specifies `opp-hz` (frequency) and `opp-microvolt` (voltage, optionally with min/max tolerance). `opp-shared` means all CPUs in a cluster share the table. Additional properties include `clock-latency-ns` (transition time), `opp-suspend` (frequency used during system suspend), `turbo-mode` (only if turbo enabled), and `opp-supported-hw` (for hw-revision filtering). The cpufreq-dt driver parses this table to configure DVFS.

**Q2: How do power domain DT bindings interact with runtime PM?**
**A:** A device references its power domain via `power-domains = <&controller DOMAIN_ID>`. During probe, the PM core attaches the device to the genpd domain. When runtime PM suspends the LAST device in a domain, genpd automatically powers off the domain (calling the provider's `power_off` callback). When any device in the domain is runtime-resumed, genpd powers the domain back on first. Domain hierarchy ensures parent domains stay on while any child domain is active.

**Q3: What is the purpose of `assigned-clocks` and `assigned-clock-rates`?**
**A:** These DT properties configure initial clock settings during device probe, before the driver runs. `assigned-clocks` lists clock handles to configure, `assigned-clock-rates` specifies target frequencies, and `assigned-clock-parents` can reparent clocks. The CCF processes these during `of_clk_set_defaults()`, called from platform device creation. This allows board-level clock configuration without driver changes.

---

## Summary

- Device tree describes PM hardware topology: power domains, clocks, regulators, OPP, thermal
- OPP tables define DVFS frequency-voltage pairs parsed by cpufreq-dt
- Power domain bindings link devices to genpd for automatic domain power management
- Clock bindings specify device clocks and initial configuration (assigned-clocks)
- Regulator bindings include suspend-state configuration for system sleep voltage
- Wakeup-source property marks devices that can wake system from sleep
- CPU idle states define C-state latency/residency for cpuidle governor decisions
- Interconnect bindings specify bandwidth requirements for NoC power management

---

[Previous Chapter: Power Management in Device Drivers ←](Chapter_16_Driver_PM.md) | [Next Chapter: Embedded System Power Management →](Chapter_18_Embedded_PM.md)
